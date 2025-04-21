---
title: "UE5 가비지 컬렉션 (1)"
excerpt: "엔진 코드를 뜯어보자"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Garbage, Collection]

permalink: /unrealengine/garbage_collection/

toc: true
toc_sticky: true

date: 2025-04-21 20:00:00
last_modified_at: 2025-04-21
---
<br>


## 다음 내용들을 다룹니다.
- 언리얼 엔진 5.44 가비지컬렉션 과정
- GCLock (FCriticalSection, FScopeLock, Atomic, Memory Barrier)
- ClusteredObjects
- ParallelFor()
- TObjectPtr<>를 권장하는 이유 (GC Barrier)
- Prefetch
- TBatchDispatcher에서 느낀 모듈화, 재사용성
- FGCObject, TStrongObjectPtr
- FPlatformMisc

<br>

### 주요 함수

ConditionalCollectGarbage()

ㄴ PerformIncrementalReachabilityAnalysis()

    ㄴ GCLock()

    ㄴ PerformReachabilityAnalysisAndConditionallyPurgeGarbage()

        ㄴ PerformReachabilityAnalysis()

            ㄴ StartReachabilityAnalysis()

                ㄴ MarkObjectsAsUnreachable()

                    ㄴ MarkClusteredObjectsAsReachable()

                    ㄴ MarkRootObjectsAsReachable()

            ㄴ PerformReachabilityAnalysisPass()

                ㄴ PerformReachabilityAnalysisOnObjects()

                    ㄴ ProcessObjectArray()

---

<br>

### 어디서 시작하는가?
`void FEngineLoop::Tick()` 에서

```cpp
void FEngineLoop::Tick()
{
	// ...
	
	// main game engine tick (world, game objects, etc.)
	GEngine->Tick(FApp::GetDeltaTime(), bIdleMode);
	
	//...
}
```

`void UGameEngine::Tick()`에서

```cpp
void UGameEngine::Tick( float DeltaSeconds, bool bIdleMode )
{
	// ...
	
	if (!bIdleMode)
	{
		SCOPE_TIME_GUARD(TEXT("UGameEngine::Tick - WorldTick"));

		// Tick the world.
		Context.World()->Tick( LEVELTICK_All, DeltaSeconds );
	}
	
	// ...
}
```

`UWorld.Tick()` 에서 

```cpp
void UWorld::Tick( ELevelTick TickType, float DeltaSeconds )
{
	// ...
	
	bInTick = false;
	Mark.Pop();

	{
		TRACE_CPUPROFILER_EVENT_SCOPE(ConditionalCollectGarbage);
		GEngine->ConditionalCollectGarbage();
	}
	
	// ...
}
```

`GEngine->ConditionalCollectGarbage()`

---

### ConditionalCollectGarbage()
<details>
<summary> 전체 코드 [ ConditionalCollectGarbage() ] </summary>
<div markdown="1">

```cpp
void UEngine::ConditionalCollectGarbage()
{
	if (GFrameCounter != LastGCFrame)
	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_ConditionalCollectGarbage);

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
		if (CVarStressTestGCWhileStreaming.GetValueOnGameThread() && IsAsyncLoading())
		{
			CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
		}
		else if (CVarForceCollectGarbageEveryFrame.GetValueOnGameThread())
		{
			CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
		}
		else
#endif
		{
			EGarbageCollectionType ForceTriggerPurge = ShouldForceGarbageCollection();
			if (ForceTriggerPurge != EGarbageCollectionType::None)
			{
				ForceGarbageCollection(ForceTriggerPurge == EGarbageCollectionType::Full);
			}

			if (bFullPurgeTriggered)
			{
				if (TryCollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true))
				{
					ForEachObjectOfClass(UWorld::StaticClass(),[](UObject* World)
					{
						CastChecked<UWorld>(World)->CleanupActors();
					});
					bFullPurgeTriggered = false;
					bShouldDelayGarbageCollect = false;
					TimeSinceLastPendingKillPurge = 0.0f;
				}
			}
			else
			{
				const bool bTestForPlayers = IsRunningDedicatedServer();
				bool bHasAWorldBegunPlay = false;
				bool bHasPlayersConnected = false;

				// Look for conditions in the worlds that would change the GC frequency
				for (const FWorldContext& Context : WorldList)
				{
					if (UWorld* World = Context.World())
					{
						if (World->HasBegunPlay())
						{
							bHasAWorldBegunPlay = true;
						}

						if (bTestForPlayers && World->NetDriver && World->NetDriver->ClientConnections.Num() > 0)
						{
							bHasPlayersConnected = true;
						}

						// If we found the conditions we wanted, no need to continue iterating
						if (bHasAWorldBegunPlay && (!bTestForPlayers || bHasPlayersConnected))
						{
							break;
						}
					}
				}

				if (bHasAWorldBegunPlay)
				{
					TimeSinceLastPendingKillPurge += FApp::GetDeltaTime();

					const float TimeBetweenPurgingPendingKillObjects = GetTimeBetweenGarbageCollectionPasses(bHasPlayersConnected);

					// See if we should delay garbage collect for this frame
					if (bShouldDelayGarbageCollect)
					{
						bShouldDelayGarbageCollect = false;
					}
					else if (IsIncrementalReachabilityAnalysisPending())
					{
						SCOPE_CYCLE_COUNTER(STAT_GCMarkTime);
						PerformIncrementalReachabilityAnalysis(GetReachabilityAnalysisTimeLimit());
					}
					// Perform incremental purge update if it's pending or in progress.
					else if (!IsIncrementalPurgePending()
						// Purge reference to pending kill objects every now and so often.
						&& (TimeSinceLastPendingKillPurge > TimeBetweenPurgingPendingKillObjects) && TimeBetweenPurgingPendingKillObjects > 0.f)
					{
						SCOPE_CYCLE_COUNTER(STAT_GCMarkTime);
						PerformGarbageCollectionAndCleanupActors();
					}
					else
					{
						SCOPE_CYCLE_COUNTER(STAT_GCSweepTime);
						float IncGCTime = GIncrementalGCTimePerFrame;
						if (GLowMemoryMemoryThresholdMB > 0.0)
						{
							float MBFree = float(PlatformMemoryHelpers::GetFrameMemoryStats().AvailablePhysical / 1024 / 1024);
#if !UE_BUILD_SHIPPING
							MBFree -= float(FPlatformMemory::GetExtraDevelopmentMemorySize() / 1024 / 1024);
#endif
							if (MBFree <= GLowMemoryMemoryThresholdMB && GLowMemoryIncrementalGCTimePerFrame > GIncrementalGCTimePerFrame)
							{
								IncGCTime = GLowMemoryIncrementalGCTimePerFrame;
							}
						}
						IncrementalPurgeGarbage(true, IncGCTime);
					}
				}
			}
		}

		if (const int32 Interval = CVarCollectGarbageEveryFrame.GetValueOnGameThread())
		{
			if (0 == (GFrameCounter % Interval))
			{
				ForceGarbageCollection(true);
			}
		}
		else if (CVarContinuousIncrementalGC.GetValueOnGameThread() > 0 && !IsIncrementalReachabilityAnalysisPending() && !IsIncrementalUnhashPending() && !IsIncrementalPurgePending())
		{
			ForceGarbageCollection(false);
		}

		LastGCFrame = GFrameCounter;
	}
	else if (IsIncrementalReachabilityAnalysisPending())
	{
		PerformIncrementalReachabilityAnalysis(GetReachabilityAnalysisTimeLimit());
	}
}
```

</div>
</details> 

```cpp
if (GFrameCounter != LastGCFrame)
{
    // ... GC 로직 ...
    LastGCFrame = GFrameCounter;
    
    if (bHasAWorldBegunPlay)
		{
			if (bShouldDelayGarbageCollect)
			{
			}
			else if (IsIncrementalReachabilityAnalysisPending())
			{
					SCOPE_CYCLE_COUNTER(STAT_GCMarkTime);
					PerformIncrementalReachabilityAnalysis(GetReachabilityAnalysisTimeLimit());
			}
		
}
else if (IsIncrementalReachabilityAnalysisPending())
{
    PerformIncrementalReachabilityAnalysis(GetReachabilityAnalysisTimeLimit());
}
```

프레임당 GC는 1회만 실행되게 합니다.  

만약 이미 해당 프레임에서 GC가 실행됐는데 또 실행 됐을 때, time limit으로 reachability analysis가 중단됐었다면 `PerformReachabilityAnalysisAndConditionallyPurgeGarbage()` 를 실행합니다.  

(`IsIncrementalReachabilityAnalysisPending()`는 기본적으로 false인데, reachability analysis가 time limit으로 중단됐을 때 true가 됩니다.)  

```cpp
#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
	if (CVarStressTestGCWhileStreaming.GetValueOnGameThread() && IsAsyncLoading())
	{
		CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
	}
	else if (CVarForceCollectGarbageEveryFrame.GetValueOnGameThread())
	{
		CollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true);
	}
	else
#endif
```

GC 스트레스 테스트 및 디버깅 용입니다.
• `CVarStressTestGCWhileStreaming`: **비동기 로딩 중 매 프레임 GC 시도**
• `CVarForceCollectGarbageEveryFrame`: **매 프레임 GC 강제 실행**

```cpp
EGarbageCollectionType ForceTriggerPurge = ShouldForceGarbageCollection();
if (ForceTriggerPurge != EGarbageCollectionType::None)
{
	ForceGarbageCollection(ForceTriggerPurge == EGarbageCollectionType::Full);
}
```

ShouldForceGarbageCollection()은 기본적으로는 `EGarbageCollectionType::None` 을 리턴합니다. 필요하면 오버라이드해서 구현하면 됩니다. 여기서 `bFullPurgeTriggered` 값이 결정됩니다.  

(메모리 부족하거나, 레벨 로딩 시간이나, 이럴 때 사용하면 될 거 같습니다.)  

```cpp
if (bFullPurgeTriggered)
{
	if (TryCollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true))
	{
		ForEachObjectOfClass(UWorld::StaticClass(),[](UObject* World)
		{
			CastChecked<UWorld>(World)->CleanupActors();
		});
		bFullPurgeTriggered = false;
		bShouldDelayGarbageCollect = false;
		TimeSinceLastPendingKillPurge = 0.0f;
	}
}
```

만약 `bFullPurgeTriggered` 이 true면, `TryCollectGarbage()`로 GC 가능할 시, 모든 UWolrd 객체를 순회하며 `CleanupActors()`를 호출합니다.  

`bFullPurgeTriggered`가 false면,

```cpp
else
{
	const bool bTestForPlayers = IsRunningDedicatedServer();
	bool bHasAWorldBegunPlay = false;
	bool bHasPlayersConnected = false;

	// Look for conditions in the worlds that would change the GC frequency
	for (const FWorldContext& Context : WorldList)
	{
		if (UWorld* World = Context.World())
		{
			if (World->HasBegunPlay())
			{
				bHasAWorldBegunPlay = true;
			}

			if (bTestForPlayers && World->NetDriver && World->NetDriver->ClientConnections.Num() > 0)
			{
				bHasPlayersConnected = true;
			}

			// If we found the conditions we wanted, no need to continue iterating
			if (bHasAWorldBegunPlay && (!bTestForPlayers || bHasPlayersConnected))
			{
				break;
			}
		}
	}
```

UWorld 객체를 순회하며,

- BeginPlay를 한 UWorld가 하나라도 있는가
- 데디케이티드 서버이거나, 데디케이티드 서버가 아니라면 월드에 접속한 클라이언트가 있는가

를 조사합니다.  

두 조건이 모두 참이 된다면, 더 이상 조사할 필요가 없기 때문에 객체 순회를 멈춥니다.  

그 후, BeginPlay를 한 월드가 있다면  

<details>
<summary> 전체 코드 [ ConditionalCollectGarbage() 후반부 ] </summary>
<div markdown="1">

```cpp
	if (bHasAWorldBegunPlay)
	{
		TimeSinceLastPendingKillPurge += FApp::GetDeltaTime();

		const float TimeBetweenPurgingPendingKillObjects = GetTimeBetweenGarbageCollectionPasses(bHasPlayersConnected);

		// See if we should delay garbage collect for this frame
		if (bShouldDelayGarbageCollect)
		{
			bShouldDelayGarbageCollect = false;
		}
		else if (IsIncrementalReachabilityAnalysisPending())
		{
			SCOPE_CYCLE_COUNTER(STAT_GCMarkTime);
			PerformIncrementalReachabilityAnalysis(GetReachabilityAnalysisTimeLimit());
		}
		// Perform incremental purge update if it's pending or in progress.
		else if (!IsIncrementalPurgePending()
			// Purge reference to pending kill objects every now and so often.
			&& (TimeSinceLastPendingKillPurge > TimeBetweenPurgingPendingKillObjects) && TimeBetweenPurgingPendingKillObjects > 0.f)
		{
			SCOPE_CYCLE_COUNTER(STAT_GCMarkTime);
			PerformGarbageCollectionAndCleanupActors();
		}
		else
		{
			SCOPE_CYCLE_COUNTER(STAT_GCSweepTime);
			float IncGCTime = GIncrementalGCTimePerFrame;
			if (GLowMemoryMemoryThresholdMB > 0.0)
			{
				float MBFree = float(PlatformMemoryHelpers::GetFrameMemoryStats().AvailablePhysical / 1024 / 1024);
#if !UE_BUILD_SHIPPING
				MBFree -= float(FPlatformMemory::GetExtraDevelopmentMemorySize() / 1024 / 1024);
#endif
				if (MBFree <= GLowMemoryMemoryThresholdMB && GLowMemoryIncrementalGCTimePerFrame > GIncrementalGCTimePerFrame)
				{
					IncGCTime = GLowMemoryIncrementalGCTimePerFrame;
				}
			}
			IncrementalPurgeGarbage(true, IncGCTime);
		}
	}
```

</div>
</details> 

```cpp
TimeSinceLastPendingKillPurge += FApp::GetDeltaTime();

const float TimeBetweenPurgingPendingKillObjects = GetTimeBetweenGarbageCollectionPasses(bHasPlayersConnected);
```

TimeSinceLastPendingKillPurge에 DeltaTime을 더해줍니다.  

TimeBetweenPurgingPendingKillObjects : PendingKillObjects를 Purging하는 주기  

이는 기본적으로 60초이고, 데디케이티드 서버의 경우 커넥트된 플레이어가 없으면 10을 곱해줍니다.  

그리고, 만약 메모리가 부족하면(low memory threshold보다 적으면) 빨리 메모리 비워야 하니깐 30초로 설정합니다.  

```cpp
// See if we should delay garbage collect for this frame
if (bShouldDelayGarbageCollect)
{
	bShouldDelayGarbageCollect = false;
}
```

만약 GC를 다음 프레임으로 미뤄야 할 경우에, 미룹니다.  

(UEngine::DelayGarbageCollection()을 통해 미룰 수 있습니다.)  

```cpp
else if (IsIncrementalReachabilityAnalysisPending())
{
	SCOPE_CYCLE_COUNTER(STAT_GCMarkTime);
	PerformIncrementalReachabilityAnalysis(GetReachabilityAnalysisTimeLimit());
}
```

Incremental Reachability Analysis은 **도달 가능성 분석을 한 번에 모두 처리하는 대신 작은 단위로 나눠서 점진적으로 수행하는 것**입니다. 이는 프레임 드랍을 막기 위해서이며, 매 프레임 제한된 시간동안만 작업을 수행하게 해서 성능을 예측 및 보장할 수 있습니다.  

`IsIncrementalReachabilityAnalysisPending()` (GIsIncrementalReachabilityPending 값 반환)는 기본적으로 false인데, reachability analysis가 time limit을 초과하는(bIsSuspended) 등의 이유로 중단되면 true가 됩니다.  

Incremental ReachabilityAnalysis의 Time Limit은 기본적으로 0.005초입니다.  

`PerformIncrementalReachabilityAnalysis()`은

```cpp
void PerformIncrementalReachabilityAnalysis(double TimeLimit)
{
	//...
	const bool bUsingTimeLimit = TimeLimit > 0.0;
	UE::GC::GReachabilityState.StartTimer(bUsingTimeLimit ? TimeLimit : 0.0);
	AcquireGCLock();	
	UE::GC::GReachabilityState.PerformReachabilityAnalysisAndConditionallyPurgeGarbage(bUsingTimeLimit);
	//...
}
```

Time Limit 계산을 위해 시작 시간을 설정하고,  

GCLock()을 얻은 후,  

`PerformReachabilityAnalysisAndConditionallyPurgeGarbage()` 을 호출합니다.  

---

### GCLock()

`GCLock()`은 

```cpp
/** Lock for GC. Will block if any other thread has locked. */
void GCLock()
{
	// Signal other threads that GC wants to run
	SetGCIsWaiting();

	// Wait until all other threads are done if they're currently holding the lock
	bool bLocked = false;
	do
	{
		FPlatformProcess::ConditionalSleep([&]()
		{
			return AsyncCounter.GetValue() == 0;
		});
		{
			FScopeLock CriticalLock(&Critical);
			if (AsyncCounter.GetValue() == 0)
			{
				GCUnlockedEvent->Reset();
				int32 GCCounterValue = GCCounter.Increment();
				check(GCCounterValue == 1); // GCLock doesn't support recursive locks
				// At this point GC can run so remove the signal that it's waiting
				FPlatformMisc::MemoryBarrier();
				ResetGCIsWaiting();
				bLocked = true;
			}
		}
	} while (!bLocked);
}
```

SetGCIsWaiting() : GCWantsToRunCounter++으로 Waiting to run으로 상태를 변경해 줍니다. 다른 thread들한테 GC 하려고 기다리고 있다고 알리는 것입니다.  

GCWantsToRunCounter는 아토믹 변수입니다.  

<details>
<summary> Atomic에 대한 간단한 개념 </summary>
<div markdown="1">

# atomic

atomic은 원자, 즉 쪼개지지 않는다는 뜻이다. 하드웨어적인 부분이다.  

예를 들어, 증가 연산을 Load ⇒ Increase ⇒ Save 이렇게 나눠서 하는 것이 아니라, 연산의 모든 과정이 한 번에 실행되어, 중간에 중단되지 않고 완벽하게 완료되거나 또는 아예 수행되지 않은 상태만을 허용하는 연산을 의미한다. 이를 통해, Data Race를 해결할 수 있다. 

## 왜 쓰는가?

1. Data Race 해결 가능
2. mutex lock으로도 data race 해결이 가능하지만, atomic이 mutex lock 보다 빠를 수 있다. (mutex는 lock 걸고 해제하고 반복하기 때문에 비교적 느릴 수 있음.)

### 성능 비교

Data Race를 해결하지 않은 예시

```cpp
**#include <iostream>
#include <mutex>
#include <thread>
#include <vector>**
#include <ctime>

**void addFunc(int& num)
{
	for (int i = 0; i < 10000; ++i)
	{
		++num;
	}
}

int main()
{**
	std::clock_t tStart = std::clock();
	
	**int num = 0;

	std::vector<std::thread> threads;
	for (int i = 0; i < 4; ++i)
	{
		threads.emplace_back(addFunc, std::ref(num));
	}

	for (auto& thread : threads)
	{
		thread.join();
	}
	std::cout << num << std::endl; // 40000이 나오지 않는다.**

	std::cout << "Time taken: " << (double)(std::clock() - tStart) / CLOCKS_PER_SEC << '\n';

	**return 0;
}**
```

0.393초가 걸린다.  

mutex lock으로 Data Race를 해결한 예시

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>
#include <ctime>

std::mutex mtx;

void addFunc(int& num)
{
	for (int i = 0; i < 10000000; ++i)
	{
		const std::lock_guard<std::mutex> lock{mtx};
		++num;
	}
}

int main()
{
	std::clock_t tStart = std::clock();

	int num = 0;

	std::vector<std::thread> threads;
	for (int i = 0; i < 4; ++i)
	{
		threads.emplace_back(addFunc, std::ref(num));
	}

	for (auto& thread : threads)
	{
		thread.join();
	}
	std::cout << num << std::endl; // 40000000이 나온다.

	std::cout << "Time taken: " << (double)(std::clock() - tStart) / CLOCKS_PER_SEC << '\n';

	return 0;
}
```

2.318초가 걸린다.  

atomic으로 Data Race를 해결한 예시

```cpp
#include <iostream>
#include <mutex>
#include <thread>
#include <vector>
#include <ctime>

std::mutex mtx;

void addFunc(std::atomic<int>& num)
{
	for (int i = 0; i < 10000000; ++i)
	{
		++num;
	}
}

int main()
{
	std::clock_t tStart = std::clock();

	std::atomic<int> num{0};

	std::vector<std::thread> threads;
	for (int i = 0; i < 4; ++i)
	{
		threads.emplace_back(addFunc, std::ref(num));
	}

	for (auto& thread : threads)
	{
		thread.join();
	}
	std::cout << num << std::endl; // 40000000이 나온다.

	std::cout << "Time taken: " << (double)(std::clock() - tStart) / CLOCKS_PER_SEC << '\n';

	return 0;
}
```

0.488초가 걸린다.  

## 주의할 점

### 연산 횟수

atomic 연산의 횟수를 고려하는 것이 중요하다.  

예를 들어,  

n += 5 는 한 번의 atomic 연산이다.  

n = n + 5 는 두 번의 atomic 연산이다. load(), store()  

### is_lock_free

사실 atomic을 하드웨어적으로 지원해 주지 않는다면, 내부적으로 mutex나 다른 lock 알고리즘을 사용한다. C++에서 정의하는 유일한 Lock Free 타입은 std::atomic_flag이다. 그 외는 하드웨어, 컴파일러에 따라 다르게 동작할 수 있다.  

Lock free가 가능한지 알아보는 방법은 is_lock_free 함수를 사용하는 것이다.

```cpp
#include <iostream>
#include <atomic>

int main()
{
	std::atomic<int> n{0};
	
	std::cout << std::boolalpha << n.is_lock_free() << std::endl;

	return 0;
}
```

is_always_lock_free 함수는 컴파일 타임에 이를 확인할 수 있게 한다.

</div>
</details>


> `TAtomic` is planned for deprecation. Please use `std::atomic`

Unreal engine은 TAtomic에서 std::atomic으로 넘어가는 것 같습니다.  


(비동기 로딩 중에 IsTimeLimitExceeded() 함수에서 IsGarbageCollectionWaiting()으로 이를 확인합니다. 시간 초과를 하지 않았더라도 가비지 컬렉션이 대기 중인 경우 시간 초과로 처리합니다.)  


FPlatformProcess::ConditionalSleep()을 통해 AsyncCounter가 0이 될 때까지 Sleep 합니다.  

(non-game thread에서 스코프 내에서 GC를 막을 때 FGCScopeGuard를 사용하는데, Scope Lock시 AsyncCounter가 올라가고, Unlock시 내려갑니다.  

GC를 막는 예시로는, 비동기 로딩 중인 UObject가 참조 추적이 잘못 되어 삭제되는 것을 막기 위해 GC 중에는 비동기 로딩을, 비동기 로딩 중에는 GC를 하지 않습니다.  

```cpp
void FGenericPlatformProcess::ConditionalSleep(TFunctionRef<bool()> Condition, float SleepTime /*= 0.0f*/)
{
	if (Condition())
	{
		return;
	}

	SCOPE_CYCLE_COUNTER(STAT_Sleep);
	FThreadIdleStats::FScopeIdle Scope;
	do
	{
		FPlatformProcess::SleepNoStats(SleepTime);
	} while (!Condition());
}
```

ConditionalSleep은 함수 파라미터로 검사할 람다 함수와 SleepTime(기본값 0.0f)을 받습니다. 현재 GC 코드에서는 SleepTime이 기본 값으로 0.0f 입니다.  

SleepNoStats(SleepTime)를 통해 프로세서 양보를 시도하면서 spin lock을 합니다. (100% busy waiting은 아닌 것)  

```cpp
void FWindowsPlatformProcess::SleepNoStats(float Seconds)
{
	uint32 Milliseconds = (uint32)(Seconds * 1000.0);
	if (Milliseconds == 0)
	{
		::SwitchToThread();
	}
	else
	{
		::Sleep(Milliseconds);
	}
}
```

Sleep()과 SwitchToThread()는 운영체제가 지원하는 함수입니다. 만약 SleepTime이 0이라면 SwitchToThread()를, 아니라면 Sleep을 호출합니다.

- Sleep()은 스레드를 Waiting 상태로 전환하고 매개변수로 준 SleepTime 후 Ready 상태로 전환합니다.
- SwitchToThread()는 현재 프로세서에서 실행할 준비가 되어 있는 다른 스레드에 실행 명령을 내립니다.
- Sleep(0.0f)와 SwitchToThread()은 둘 다 양보를 하는 함수지만, 차이점은 Sleep은 자기와 같거나 자기보다 높은 우선순위를 가진 스레드가 없다면 Context Switch를 하지 않는다는 것이고, SwitchToThread()는 **우선순위와 무관하게** Context Switch를 시도한다는 것입니다.

<div>
    <img src="/assets/images/posts_img/garbagecollection/image1.png" alt="threadstate" width="75%" min-width="100px" itemprop="image">
</div>

절전 모드 함수 : [https://learn.microsoft.com/ko-kr/windows/win32/api/synchapi/nf-synchapi-sleep?redirectedfrom=MSDN](https://learn.microsoft.com/ko-kr/windows/win32/api/synchapi/nf-synchapi-sleep?redirectedfrom=MSDN)  
SwitchToThread 함수 : [https://learn.microsoft.com/ko-kr/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread?redirectedfrom=MSDN](https://learn.microsoft.com/ko-kr/windows/win32/api/processthreadsapi/nf-processthreadsapi-switchtothread?redirectedfrom=MSDN)

<br>

AsyncCounter가 0이 되어서 ConditionalSleep()가 끝나면,

```cpp
FCriticalSection Critical;
//...
	
{
	FScopeLock CriticalLock(&Critical);
	if (AsyncCounter.GetValue() == 0)
	{
		GCUnlockedEvent->Reset();
		int32 GCCounterValue = GCCounter.Increment();
		check(GCCounterValue == 1); // GCLock doesn't support recursive locks
		// At this point GC can run so remove the signal that it's waiting
		FPlatformMisc::MemoryBarrier();
		ResetGCIsWaiting();
		bLocked = true;
	}
}
```

FScopeLock을 통해 Race Condition을 막습니다.  
FScopeLock은 C++의 std::lock_guard과 유사한 스코프 벗어나면 알아서 Unlock() 하는 Unreal의 RAII 기반 동기화 도구입니다. std::mutex가 아니라 언리얼엔진 내 FCriticalSection과 함께 사용합니다.  

AsyncCounter를 재확인하고,  
GC를 Unlock할 때, GCUnlockedEvent를 트리거 하기 위해서 GCUnlockedEvent를 리셋해 주고,  
GC가 실행 중인(running) 상태를 나타내는 카운터(GCCounterValue)를 증가시키고,  

이제 GC를 Lock했고, 이는 GC가 실행 가능한 상태라는 것이기 때문에 ResetGCIsWaiting()으로 GC가 Waiting to run이 아니라고 설정하고, bLocked를 true로 설정합니다.  

근데 여기서 GCCounter를 올리는 것과 ResetGCIsWaiting() 둘의 순서는 바뀌면 안 되기 때문에 (CPU나 컴파일러가 임의로 순서를 바꿀 수도 있습니다.) 메모리 베리어를 통해 둘의 순서를 보장해 줍니다. 

---

### PerformReachabilityAnalysisAndConditionallyPurgeGarbage()
<details>
<summary> 전체 코드 [ PerformReachabilityAnalysisAndConditionallyPurgeGarbage() ] </summary>
<div markdown="1">

```cpp
void FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage(bool bReachabilityUsingTimeLimit)
{
	using namespace UE::GC::Private;

	LLM_SCOPE(ELLMTag::GC);

	if (!GIsIncrementalReachabilityPending)
	{
		GGCStats = UE::GC::Private::FStats();
		GGCStats.bInProgress = true;
		GGCStats.bStartedAsFullPurge = bPerformFullPurge;
		GGCStats.NumObjects = GUObjectArray.GetObjectArrayNumMinusAvailable() - GUObjectArray.GetFirstGCIndex();
		GGCStats.NumClusters = GUObjectClusters.GetNumAllocatedClusters();
		GGCStats.ReachabilityTimeLimit = GetReachabilityAnalysisTimeLimit();
	}
	GGCStats.bFinishedAsFullPurge = bPerformFullPurge;

	if (bPerformFullPurge)
	{
		UE::GC::PreCollectGarbageImpl<true>(ObjectKeepFlags);
	}
	else
	{
		UE::GC::PreCollectGarbageImpl<false>(ObjectKeepFlags);
	}	
	
	const bool bForceNonIncrementalReachability =
		!GIsIncrementalReachabilityPending &&
		(bPerformFullPurge || !GAllowIncrementalReachability);

	{
		// When incremental reachability is enabled we start the timer before acquiring GC lock
		// and here we keep track of reference processing time only
		const double ReferenceProcessingStartTime = FPlatformTime::Seconds();
		// When performing the first iteration of reachability analysis start the timer when we actually start processing
		// iteration as we don't have control over various callbacks being fired in PreCollectGarbageImpl and can't be responsible for any hitches in them
		if (IterationStartTime == 0.0)
		{
			IterationStartTime = ReferenceProcessingStartTime;
			IterationTimeLimit = bReachabilityUsingTimeLimit ? GIncrementalReachabilityTimeLimit : 0.0;
		}

		SCOPED_NAMED_EVENT(FRealtimeGC_PerformReachabilityAnalysis, FColor::Red);
		DECLARE_SCOPE_CYCLE_COUNTER(TEXT("FRealtimeGC::PerformReachabilityAnalysis"), STAT_FArchiveRealtimeGC_PerformReachabilityAnalysis, STATGROUP_GC);

		if (bForceNonIncrementalReachability)
		{
			IncrementalMarkPhaseTotalTime = 0.0;
			ReferenceProcessingTotalTime = 0.0;
			PerformReachabilityAnalysis();
		}
		else
		{
			if (!GIsIncrementalReachabilityPending)
			{
				ReferenceProcessingTotalTime = 0.0;
				IncrementalMarkPhaseTotalTime = 0.0;
			}

			PerformReachabilityAnalysis();
		}

		GIsIncrementalReachabilityPending = GReachabilityState.IsSuspended();

		const double CurrentTime = FPlatformTime::Seconds();
		const double ReferenceProcessingElapsedTime = CurrentTime - ReferenceProcessingStartTime;
		const double ElapsedTime = CurrentTime - IterationStartTime;
		ReferenceProcessingTotalTime += ReferenceProcessingElapsedTime;
		IncrementalMarkPhaseTotalTime += ElapsedTime;

		GGCStats.ReachabilityTime += ReferenceProcessingElapsedTime;
		GGCStats.TotalTime += ReferenceProcessingElapsedTime;

		if (UE_LOG_ACTIVE(LogGarbage, Log))
		{
			if (GIsIncrementalReachabilityPending)
			{
				const double SuspendLatency = FMath::Max(0.0, CurrentTime - (IterationStartTime + IterationTimeLimit));
				UE_LOG(LogGarbage, Log, TEXT("GC Reachability iteration time: %.2f ms (%.2f ms on reference traversal, latency: %.3f ms)"), ElapsedTime * 1000, ReferenceProcessingElapsedTime * 1000, SuspendLatency * 1000.0);
			}
			else
			{
				const double ReferenceProcessingTotalTimeMs = ReferenceProcessingTotalTime * 1000;
				const double IncrementalMarkPhaseTotalTimeMs = IncrementalMarkPhaseTotalTime * 1000;
				UE_LOG(LogGarbage, Log, TEXT("GC Reachability Analysis total time: %.2f ms (%.2f ms on reference traversal)"), IncrementalMarkPhaseTotalTimeMs, ReferenceProcessingTotalTimeMs);
				FString ExtraDetail = WITH_VERSE_VM ? FString::Printf(TEXT("and %d verse cells"), Stats.NumVerseCells) : FString();
				UE_LOG(LogGarbage, Log, TEXT("%.2f ms for %sGC - %d refs/ms while processing %d references from %d objects %s with %d clusters"),
					IncrementalMarkPhaseTotalTimeMs,
					bForceNonIncrementalReachability ? TEXT("") : TEXT("Incremental "),
					(int32)(Stats.NumReferences / ReferenceProcessingTotalTimeMs),
					Stats.NumReferences,
					Stats.NumObjects,
					*ExtraDetail,
					GUObjectClusters.GetNumAllocatedClusters());
			}
		}

		// Reset timer and the time limit. These values will be set to their target values in the next iteration but we don't want 
		// them to be set before the debug reachability run below
		IterationTimeLimit = 0.0;
		IterationStartTime = 0.0;
	}

	if (!GIsIncrementalReachabilityPending && Stats.bFoundGarbageRef && GGarbageReferenceTrackingEnabled > 0)
	{
		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(GarbageCollectionDebug);
		SCOPED_NAMED_EVENT(FRealtimeGC_PerformReachabilityAnalysisRerun, FColor::Orange);
		DECLARE_SCOPE_CYCLE_COUNTER(TEXT("FRealtimeGC::PerformReachabilityAnalysisRerun"), STAT_FArchiveRealtimeGC_PerformReachabilityAnalysisRerun, STATGROUP_GC);		
		const double StartTime = FPlatformTime::Seconds();
		{
			// If we are going to scan again, we need to cycle verse GC so the cells are unmarked.
			StopVerseGC();
			TGuardValue<bool> GuardReachabilityUsingTimeLimit(bReachabilityUsingTimeLimit, false);
			FRealtimeGC GC;
			GC.Stats = Stats; // This is to pass Stats.bFoundGarbageRef to CG
			GC.PerformReachabilityAnalysis(ObjectKeepFlags, GetReferenceCollectorOptions(bPerformFullPurge));
		}
		const double ElapsedTime = FPlatformTime::Seconds() - StartTime;
		GGCStats.GarbageTrackingTime = ElapsedTime;
		GGCStats.TotalTime += ElapsedTime;
		UE_LOG(LogGarbage, Log, TEXT("%.2f ms for GC rerun to track garbage references (gc.GarbageReferenceTrackingEnabled=%d)"), ElapsedTime * 1000, GGarbageReferenceTrackingEnabled);
	}
	// Maybe purge garbage (if we're done with incremental reachability and there's still time left)

	if (bPerformFullPurge)
	{
		UE::GC::PostCollectGarbageImpl<true>(ObjectKeepFlags);
	}
	else
	{
		UE::GC::PostCollectGarbageImpl<false>(ObjectKeepFlags);
	}
}
```

</div>
</details> 

```cpp
if (!GIsIncrementalReachabilityPending)
{
	GGCStats = UE::GC::Private::FStats();
	GGCStats.bInProgress = true;
	GGCStats.bStartedAsFullPurge = bPerformFullPurge;
	GGCStats.NumObjects = GUObjectArray.GetObjectArrayNumMinusAvailable() - GUObjectArray.GetFirstGCIndex();
	GGCStats.NumClusters = GUObjectClusters.GetNumAllocatedClusters();
	GGCStats.ReachabilityTimeLimit = GetReachabilityAnalysisTimeLimit();
}
GGCStats.bFinishedAsFullPurge = bPerformFullPurge;
```

만약 GIsIncrementalReachabilityPending가 아니라면, 즉, 이미 진행 중이었던 GC가 아니라 GC의 시작이면 GCStats를 초기화합니다.  
bFinishedAsFullPurge 값은 GIsIncrementalReachabilityPending와 상관없이 대입해 줍니다. (false였다가 true일 수 있으니)

```cpp
if (bPerformFullPurge)
{
	UE::GC::PreCollectGarbageImpl<true>(ObjectKeepFlags);
}
else
{
	UE::GC::PreCollectGarbageImpl<false>(ObjectKeepFlags);
}
```

bPerformFullPurge 값에 따라 PreCollectGarbageImpl()를 수행합니다.  

PreCollectGarbageImpl()은 이런 작업을 합니다.

- GFlushStreamingOnGC 설정을 켜 둔 경우, GIsIncrementalReachabilityPending가 false이고, 비동기 로딩 중이라면, 비동기 로딩 중이었던 것들을 모두 TimeLimit을 false로 해서 GC 중 비동기 로딩 중이던 UObject가 참조 추적이 잘못 되어 삭제되는 것을 방지합니다.
- GetPreGarbageCollectDelegate().Broadcast()
- IncrementalPurge가 보류/진행 중이었다면 TimeLimit을 false로 해서 끝낸다.
- LockUObjectHashTables()으로 GC 중 UObject 객체 추가/제거 방지
- GC 무시 객체, 객체 플래그 등의 검증

```cpp
const bool bForceNonIncrementalReachability =
	!GIsIncrementalReachabilityPending &&
	(bPerformFullPurge || !GAllowIncrementalReachability);
```

강제 비증분 GC인지 확인합니다.

```cpp
if (bForceNonIncrementalReachability)
{
	IncrementalMarkPhaseTotalTime = 0.0;
	ReferenceProcessingTotalTime = 0.0;
	PerformReachabilityAnalysis();
}
else
{
	if (!GIsIncrementalReachabilityPending)
	{
		ReferenceProcessingTotalTime = 0.0;
		IncrementalMarkPhaseTotalTime = 0.0;
	}

	PerformReachabilityAnalysis();
}
```

만약 증분 GC가 진행 중이었다면 TotalTime을 초기화하지 않습니다.  
PerformReachabilityAnalysis()으로 ReachabilityAnalysis을 시작합니다.

```cpp
void FReachabilityAnalysisState::PerformReachabilityAnalysis()
{
	if (!bIsSuspended)
	{
		Init();
		NumRechabilityIterationsToSkip = FMath::Max(0, GDelayReachabilityIterations);
	}

	if (bPerformFullPurge)
	{
		UE::GC::CollectGarbageFull(ObjectKeepFlags);
	}
	else if (NumRechabilityIterationsToSkip == 0 || // Delay reachability analysis by NumRechabilityIterationsToSkip (if desired)
		!bIsSuspended || // but only but only after the first iteration (which also does MarkObjectsAsUnreachable)
		IterationTimeLimit <= 0.0f) // and only when using time limit (we're not using the limit when we're flushing reachability analysis when starting a new one or on exit)
	{
		UE::GC::CollectGarbageIncremental(ObjectKeepFlags);
	}
	else
	{
		--NumRechabilityIterationsToSkip;
	}

	FinishIteration();
}
```

bIsSuspended는 timelimit을 초과해서 도중에 종료되었는지를 나타냅니다.  
도중에 종료된 게 아니라 새로운 ReachabilityAnalysis면 Init을 해 줍니다.  
GDelayReachabilityIterations를 통해 ReachabilityAnalysis 미루는 횟수를 설정할 수 있습니다. (첫 반복은 미뤄지지 않습니다)  

만약 bPerformFullPurge라면 CollectGarbageFull  
아니라면 GDelayReachabilityIterations만큼 미룬 이후 또는 GC의 첫 반복인 경우 또는 TimeLimit이 없는 경우  CollectGarbageIncremental()  
모두 아니라면 미룹니다.  
 
그 후, FinishIteration()을 통해 NumIterations++ 합니다.  

CollectGarbageFull()이면 CollectGarbageImpl<true>(KeepFlags), 과 CollectGarbageIncremental()이면 CollectGarbageImpl<false>(KeepFlags)을 실행합니다.  

```cpp
template<bool bPerformFullPurge>
void CollectGarbageImpl(EObjectFlags KeepFlags)
{
	{
		// Reachability analysis.
		{
			const EGCOptions Options = GetReferenceCollectorOptions(bPerformFullPurge);

			// Perform reachability analysis.
			FRealtimeGC GC;
			GC.PerformReachabilityAnalysis(KeepFlags, Options);
		}
	}
}
```

```cpp
EGCOptions GetReferenceCollectorOptions(bool bPerformFullPurge)
{
	return
		// Fall back to single threaded GC if processor count is 1 or parallel GC is disabled
		// or detailed per class gc stats are enabled (not thread safe)
		(ShouldForceSingleThreadedGC() ? EGCOptions::None : EGCOptions::Parallel) |
		// Toggle between Garbage Eliination enabled or disabled
		(UObjectBaseUtility::IsGarbageEliminationEnabled() ? EGCOptions::EliminateGarbage : EGCOptions::None) |
		// Toggle between Incremental Reachability enabled or disabled
		((GAllowIncrementalReachability && !bPerformFullPurge) ? EGCOptions::IncrementalReachability : EGCOptions::None);
}
```

멀티스레드 GC인지, Garbage를 GC가 자동으로 제거할지, IncrementalReachability인지를 Options에 넣고, PerformReachabilityAnalysis()를 실행합니다.  

그 후, FRealtimeGC 객체를 만들어서 플래그와 GC옵션에 맞게 PerformReachabilityAnalysis()로 ReachabilityAnalysis를 수행합니다. (~~아까도 PerformReachabilityAnalysis()였지만!~~)  

### PerformReachabilityAnalysis()

<details>
<summary> 전체 코드 [ PerformReachabilityAnalysis() ] </summary>
<div markdown="1">

```cpp
	/**
	 * Performs reachability analysis.
	 *
	 * @param KeepFlags		Objects with these flags will be kept regardless of being referenced or not
	 */
	void PerformReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
	{
		LLM_SCOPE(ELLMTag::GC);

		const bool bIsGarbageTracking = !GReachabilityState.IsSuspended() && Stats.bFoundGarbageRef;

		if (!GReachabilityState.IsSuspended())
		{
			StartReachabilityAnalysis(KeepFlags, Options);
			// We start verse GC here so that the objects are unmarked prior to verse marking them
			StartVerseGC();
		}

		{
			const double StartTime = FPlatformTime::Seconds();

			do
			{
				PerformReachabilityAnalysisPass(Options);
			// NOTE: It is critical that VerseGCActive is called prior to checking GReachableObjects.  While VerseGCActive is true,
			// items can still be added to GReachableObjects.  So if reversed, during the point where GReachableObjects is checked
			// and VerseGCActive returns false, something might have been marked.  Reversing insures that Verse will not add anything 
			// if Verse is no longer active.
			} while ((VerseGCActive() || !Private::GReachableObjects.IsEmpty() || !Private::GReachableClusters.IsEmpty()) && !GReachabilityState.IsSuspended());

			const double ElapsedTime = FPlatformTime::Seconds() - StartTime;
			if (!bIsGarbageTracking)
			{
				GGCStats.ReferenceCollectionTime += ElapsedTime;
			}
			UE_LOG(LogGarbage, Verbose, TEXT("%f ms for Reachability Analysis"), ElapsedTime * 1000);
		}

PRAGMA_DISABLE_DEPRECATION_WARNINGS
		// Allowing external systems to add object roots. This can't be done through AddReferencedObjects
		// because it may require tracing objects (via FGarbageCollectionTracer) multiple times
		if (!GReachabilityState.IsSuspended())
		{
			FCoreUObjectDelegates::TraceExternalRootsForReachabilityAnalysis.Broadcast(*this, KeepFlags, !(Options & EGCOptions::Parallel));
		}
PRAGMA_ENABLE_DEPRECATION_WARNINGS
	}
```

</div>
</details> 

```cpp
if (!GReachabilityState.IsSuspended())
{
	StartReachabilityAnalysis(KeepFlags, Options);
	// We start verse GC here so that the objects are unmarked prior to verse marking them
	StartVerseGC();
}
```

GC TimeLimit 초과로 중단되었던 거 아니면, StartReachabilityAnalysis()와 StartVerseGC()을 수행합니다.  
VerseGC는 넘기도록 하겠습니다.  
**“Verse** 는 **에픽게임즈** 가 개발한 프로그래밍 언어로, **포크리** 디바이스를 커스터마이징하는 등 **포트나이트 언리얼 에디터** 에서 나만의 게임플레이를 제작하는 데 사용할 수 있는 언어입니다.”  

---

### StartReachabilityAnalysis()

**이 함수에서 모든 객체를 잠재적 도달 불가능 상태로 초기화하고, 클러스터 객체와 루트 객체 도달 가능 상태로 설정합니다.**  

StartReachabilityAnalysis() : 각종 값들 초기화 (InitialReferences, InitialObjects, InitialCollection, GObjectCountDuringLastMarkPhase) 후에 MarkObjectsAsUnreachable() 수행.  

- InitialObjects : Mark 과정에서 탐색을 시작할 첫 오브젝트 (예 : Root Object)


### MarkObjectsAsUnreachable()

<details>
<summary> 전체 코드 [ MarkObjectsAsUnreachable() ] </summary>
<div markdown="1">

```cpp
FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	using namespace UE::GC;

	// Don't swap the flags if we're re-entering this function to track garbage references
	if (const bool bInitialMark = !Stats.bFoundGarbageRef)
	{
		// This marks all UObjects as MaybeUnreachable
		Swap(GReachableObjectFlag, GMaybeUnreachableObjectFlag);
	}

	// Not counting the disregard for GC set to preserve legacy behavior
	GObjectCountDuringLastMarkPhase.Set(GUObjectArray.GetObjectArrayNumMinusAvailable() - GUObjectArray.GetFirstGCIndex());

	EGatherOptions GatherOptions = GetObjectGatherOptions();

	// Now make sure all clustered objects and root objects are marked as Reachable. 
	// This could be considered as initial part of reachability analysis and could be made incremental.
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	MarkRootObjectsAsReachable(GatherOptions, KeepFlags, InitialObjects);
}
```

</div>
</details> 

```cpp
// Don't swap the flags if we're re-entering this function to track garbage references
if (const bool bInitialMark = !Stats.bFoundGarbageRef)
{
	// This marks all UObjects as MaybeUnreachable
	Swap(GReachableObjectFlag, GMaybeUnreachableObjectFlag);
}
```

아직 가비지 참조가 발견되지 않은 초기 마크 단계라면 모든 객체를 **잠재적 도달 불가능 상태로 초기화**합니다.  

```cpp
GObjectCountDuringLastMarkPhase.Set(GUObjectArray.GetObjectArrayNumMinusAvailable() - GUObjectArray.GetFirstGCIndex());
```

조사해야 하는 대상 객체 수를 설정해 주고,  

```cpp
EGatherOptions GatherOptions = GetObjectGatherOptions();
```

UnreachableObject 모으는 것을 싱글스레드로 or 멀티스레드로 할지 옵션 설정해 주고,  

```cpp
	// Now make sure all clustered objects and root objects are marked as Reachable. 
	// This could be considered as initial part of reachability analysis and could be made incremental.
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	MarkRootObjectsAsReachable(GatherOptions, KeepFlags, InitialObjects);
```

클러스터 객체와 루트 객체를 Reachable로 마크합니다. 그리고 InitialObjects에 추가합니다.  
클러스터 객체란? : 관련된 UObject들을 클러스터라는 단위로 묶은 것입니다. 이를 사용하면 Cluster의 Root를 Reachable Mark를 하게 되면 클러스터에 속한 UObject들도 모두 Reachable Mark 하는 방법으로 가비지 콜렉션의 마크 단계를 최적화할 수 있습니다. (개별 액터 추적이 아니라 클러스터 단위 한 번에 추적)  

MarkClusteredObjectsAsReachable() 과정 :  
GUObjectClusters에 FUObjectCluster 배열이 있고, 각 FUObjectCluster는 그 클러스터의 Root Object의 GUObjectArray에서의 Index인 RootIndex를 가지고 있습니다. GUObjectArray에서 해당 클러스터의 RootItem을 가져와서 Garbage인지 확인합니다. Garbage라면 클러스터를 없앱니다. Garbage가 아니라면, 모두 Reachable Mark를 합니다. 또한, 클러스터 안에 RootFlags를 가진 객체가 하나라도 있으면 이 클러스터에 참조되는 클러스터들(ReferencedClusters)을 Reachable Mark합니다.  

MarkClusteredObjectsAsReachable()와 MarkRootObjectsAsReachable() 모두 ParallelFor()을 사용하여 작업을 병렬로 처리합니다.  

MarkRootObjectsAsReachable() 코드 일부를 보면,  

```cpp
		using FMarkRootsState = TThreadedGather<TArray<UObject*>>;
		
		FMarkRootsState MarkRootsState;		

		{
			GRootsCritical.Lock();
			TArray<int32> RootsArray(GRoots.Array());				
			GRootsCritical.Unlock();
			MarkRootsState.Start(Options, RootsArray.Num());
			FMarkRootsState::FThreadIterators& ThreadIterators = MarkRootsState.GetThreadIterators();

			ParallelFor(TEXT("GC.MarkRootObjectsAsReachable"), MarkRootsState.NumWorkerThreads(), 1, [&ThreadIterators, &RootsArray](int32 ThreadIndex)
			{
				TRACE_CPUPROFILER_EVENT_SCOPE(MarkClusteredObjectsAsReachableTask);
				FMarkRootsState::FIterator& ThreadState = ThreadIterators[ThreadIndex];

				while (ThreadState.Index <= ThreadState.LastIndex)
				{
					FUObjectItem* RootItem = &GUObjectArray.GetObjectItemArrayUnsafe()[RootsArray[ThreadState.Index++]];
					UObject* Object = static_cast<UObject*>(RootItem->Object);

					// IsValidLowLevel is extremely slow in this loop so only do it in debug
					checkSlow(Object->IsValidLowLevel());					
#if DO_GUARD_SLOW
					// We cannot mark Root objects as Garbage.
					checkCode(if (RootItem->HasAllFlags(EInternalObjectFlags::Garbage | EInternalObjectFlags::RootSet)) { UE_LOG(LogGarbage, Fatal, TEXT("Object %s is part of root set though has been marked as Garbage!"), *Object->GetFullName()); });
#endif

					RootItem->FastMarkAsReachableInterlocked_ForGC();
					ThreadState.Payload.Add(Object);
				}
			}, (MarkRootsState.NumWorkerThreads() == 1) ? EParallelForFlags::ForceSingleThread : EParallelForFlags::None);			
		}
```


Root Object 배열을 가져오고, `MarkRootsState.Start(Options, RootsArray.Num())`를 통해 병렬로 나눌 스레드 개수를 정하고, 각 스레드가 처리할 Root Object 배열의 시작 인덱스와 끝 인덱스를 나눕니다.  
그 후, MarkRootsState.NumWorkerThreads() 만큼 스레드를 나눠서 Root Object Mark 작업을 처리합니다. (만약 1개라면 ForceSingleThread)  

그리고, MarkRootObjectsAsReachable()는 Editor 환경에서는 GUObjectArray에 속한 모든 UObject를 검사합니다. Root Object가 아니고, 제거될 가비지(bWithGarbageElimination && IsGarbage)가 아니면서 , EObjectFlags에 RF_Standalone이 있을 경우 Reachable Mark합니다. **즉, 에디터 환경에서는 이 과정에서 모든 UObject를 순회하기 때문에 GC가 훨 느립니다**.  

---

```cpp
do
{
	PerformReachabilityAnalysisPass(Options);
// NOTE: It is critical that VerseGCActive is called prior to checking GReachableObjects.  While VerseGCActive is true,
// items can still be added to GReachableObjects.  So if reversed, during the point where GReachableObjects is checked
// and VerseGCActive returns false, something might have been marked.  Reversing insures that Verse will not add anything 
// if Verse is no longer active.
} while ((VerseGCActive() || !Private::GReachableObjects.IsEmpty() || !Private::GReachableClusters.IsEmpty()) && !GReachabilityState.IsSuspended());
```

### PerformReachabilityAnalysisPass()

<details>
<summary> 전체 코드 [ PerformReachabilityAnalysisPass() ] </summary>
<div markdown="1">

```cpp
void PerformReachabilityAnalysisPass(const EGCOptions Options)
{
	FContextPoolScope Pool;
	FWorkerContext* Context = nullptr;
	const bool bIsSingleThreaded = !(Options & EGCOptions::Parallel);

	if (GReachabilityState.IsSuspended())
	{
		Context = GReachabilityState.GetContextArray()[0];
		Context->bDidWork = false;
		InitialObjects.Reset();
	}
	else
	{
		Context = Pool.AllocateFromPool();
		if (bIsSingleThreaded)
		{
			GReachabilityState.SetupWorkers(1);
			GReachabilityState.GetContextArray()[0] = Context;
		}
	}

	if (!Private::GReachableObjects.IsEmpty())
	{
		// Add objects marked with the GC barrier to the inital set of objects for the next iteration of incremental reachability
		Private::GReachableObjects.PopAllAndEmpty(InitialObjects);
		GGCStats.NumBarrierObjects += InitialObjects.Num();
		UE_LOG(LogGarbage, Verbose, TEXT("Adding %d object(s) marker by GC barrier to the list of objects to process"), InitialObjects.Num());
		ConditionallyAddBarrierReferencesToHistory(*Context);
	}
	else if (GReachabilityState.GetNumIterations() == 0 || (Stats.bFoundGarbageRef && !GReachabilityState.IsSuspended()))
	{
		Context->InitialNativeReferences = GetInitialReferences(Options);
	}

	if (!Private::GReachableClusters.IsEmpty())
	{
		// Process cluster roots that were marked as reachable by the GC barrier
		TArray<FUObjectItem*> KeepClusterRefs;
		Private::GReachableClusters.PopAllAndEmpty(KeepClusterRefs);
		for (FUObjectItem* ObjectItem : KeepClusterRefs)
		{
			// Mark referenced clusters and mutable objects as reachable
			MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), InitialObjects);
		}
	}

	Context->SetInitialObjectsUnpadded(InitialObjects);

	PerformReachabilityAnalysisOnObjects(Context, Options);

	if (!GReachabilityState.CheckIfAnyContextIsSuspended())
	{
		GReachabilityState.ResetWorkers();
		Stats.AddStats(Context->Stats);
		GReachabilityState.UpdateStats(Context->Stats);
		Pool.ReturnToPool(Context);
	}
	else if (bIsSingleThreaded)
	{
		Context->ResetInitialObjects();
		Context->InitialNativeReferences = TConstArrayView<UObject**>();
	}
}
```

</div>
</details> 

```cpp
if (GReachabilityState.IsSuspended())
{
	Context = GReachabilityState.GetContextArray()[0];
	Context->bDidWork = false;
	InitialObjects.Reset();
}
else
{
	Context = Pool.AllocateFromPool();
	if (bIsSingleThreaded)
	{
		GReachabilityState.SetupWorkers(1);
		GReachabilityState.GetContextArray()[0] = Context;
	}
}
```

중단됐었다면 이전 것을 재개하고, 아니라면 새로 시작합니다.

```cpp
if (!Private::GReachableObjects.IsEmpty())
{
	// Add objects marked with the GC barrier to the inital set of objects for the next iteration of incremental reachability
	Private::GReachableObjects.PopAllAndEmpty(InitialObjects);
	GGCStats.NumBarrierObjects += InitialObjects.Num();
	UE_LOG(LogGarbage, Verbose, TEXT("Adding %d object(s) marker by GC barrier to the list of objects to process"), InitialObjects.Num());
	ConditionallyAddBarrierReferencesToHistory(*Context);
}
else if (GReachabilityState.GetNumIterations() == 0 || (Stats.bFoundGarbageRef && !GReachabilityState.IsSuspended()))
{
	Context->InitialNativeReferences = GetInitialReferences(Options);
}

if (!Private::GReachableClusters.IsEmpty())
{
	// Process cluster roots that were marked as reachable by the GC barrier
	TArray<FUObjectItem*> KeepClusterRefs;
	Private::GReachableClusters.PopAllAndEmpty(KeepClusterRefs);
	for (FUObjectItem* ObjectItem : KeepClusterRefs)
	{
		// Mark referenced clusters and mutable objects as reachable
		MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), InitialObjects);
	}
}
```

if문은 GC Barrier( UObject::MarkAsReachable() ) 와 관련된 코드입니다. GC가 객체 메모리를 회수하기 전에, 실행 중인 코드가 객체 참조를 변경할 때 이를 추적해서 안전하게 GC 대상에서 제외시키는 메커니즘입니다. GReachableObjects, GReachableClusters에 들어갑니다. **즉시 객체를 reachable로 만들 때 사용하며, 이를 통해 Incremental GC 도중에도 참조가 변경된 객체를 누락 없이 마킹할 수 있습니다.** InitialObjects에 추가됩니다.  

// 이 부분 왜 else if에 있는지?  
else if문은 InitialNativeReferences에 InitialReferences를 넣는 코드입니다.  
GetInitialReferences()를 쓰면 AddInitialReferences()를 써서 InitialReferencedObjects들을 Collector에 등록하고 InitialReferences에 추가합니다. 이걸 InitialNativeReferences에 대입합니다.  

```cpp
explicit FORCEINLINE FObjectPtr(UObject* Object)
	: Handle(UE::CoreUObject::Private::MakeObjectHandle(Object))
{
#if UE_OBJECT_PTR_GC_BARRIER
	ConditionallyMarkAsReachable(Object);
#endif // UE_OBJECT_PTR_GC_BARRIER
}
```

TObjectPtr<>을 쓰라고 하는 이유 중 하나가 GC Barrier입니다. TObjectPtr은 할당 시 MarkAsReachable을 통해 Incremental GC에서 누락되지 않게 합니다.  

이 코드에서 UObject::MarkAsReachable()로 GC에서 제외한 GReachableObjects와 GReachableClusters들을 Mark해 줍니다.  

---

```cpp
Context->SetInitialObjectsUnpadded(InitialObjects);
```

```cpp
/** @param Objects must outlive this context. It's data is padded by repeating the last object to allow prefetching past the end. */
void SetInitialObjectsUnpadded(TArray<UObject*>& Objects)
{
	PadObjectArray(Objects);
	SetInitialObjectsPrepadded(Objects);
}
```

이 함수는 Prefetch 최적화를 위해 Padding을 하는 것입니다. InitialObjects 배열의 끝 이후 메모리에 배열의 마지막 요소를 몇 개 넣습니다. 배열의 끝을 넘어 prefetching 하는 것을 가능하게 합니다.  

Prefetch와 Padding  

- 메모리 접근 지연을 줄이기 위해 아직 사용되기 전인 데이터를 미리 캐시 계층에 올리는 것입니다.
- 배열을 순회하면서 고정된 거리(PREFETCH_DISTANCE)만큼 앞서서 미리 가져오도록 할 때(prefetch), 루프가 배열의 끝에 가까워지면 i + PREFETCH_DISTANCE가 배열 범위를 벗어나게 됩니다. 이때 메모리 할당을 좀 더 크게 잡고, 마지막 유효 요소를 추가로 복제해 두면 넘어간 위치에 대해서도 안전하게 prefetch할 수 있습니다.
- 만약 Padding을 안 하고 배열 범위를 벗어나지 않게 분기문을 쓰는 것은 비용이 늘어나 성능이 저하될 수 있어서 Padding을 한다고 합니다.


---

```cpp
PerformReachabilityAnalysisOnObjects(Context, Options);
```

```cpp
virtual void PerformReachabilityAnalysisOnObjects(FWorkerContext* Context, EGCOptions Options) override
{
	(this->*ReachabilityAnalysisFunctions[GetGCFunctionIndex(Options)])(*Context);
}
```

GC 옵션(Parallel, EliminateGarbage, IncrementalReachability)에 해당하는 ReachabilityAnalysisFunction을 호출합니다.  

호출되는 함수는 FRealtimeGC::PerformReachabilityAnalysisOnObjectsInternal<Options> 입니다.  

```cpp
template <EGCOptions Options>
void PerformReachabilityAnalysisOnObjectsInternal(FWorkerContext& Context)
{
	// ...
	TReachabilityProcessor<Options> Processor;
	CollectReferencesForGC<TReachabilityCollector<Options>>(Processor, Context);
}
```

Processor는 TReachabilityProcessor<Options> 입니다.  

```cpp
template<class CollectorType, class ProcessorType>
FORCEINLINE void CollectReferencesForGC(ProcessorType& Processor, UE::GC::FWorkerContext& Context)
{
	using FastReferenceCollector = TFastReferenceCollector<ProcessorType, CollectorType>;

	if constexpr (IsParallel(ProcessorType::Options))
	{
		ProcessAsync([](void* P, FWorkerContext& C) { FastReferenceCollector(*reinterpret_cast<ProcessorType*>(P)).ProcessObjectArray(C); }, &Processor, Context);
	}
	else
	{
		FastReferenceCollector(Processor).ProcessObjectArray(Context);
	}
}
```

FastReferenceCollector(Processor).ProcessObjectArray(Context)를 수행합니다. (IsParallel이면 ProcessAsync()로)  

<details>
<summary> ProcessAsync() [ GarbageCollection.cpp ] </summary>
<div markdown="1">

```cpp
TArrayView<FWorkerContext*> Contexts = InitializeAsyncProcessingContexts(InContext);
TSharedRef<FWorkCoordinator> WorkCoordinator = MakeShared<FWorkCoordinator>(Contexts, FTaskGraphInterface::Get().GetNumWorkerThreads());
for (FWorkerContext* Context : Contexts)
{
	Context->bDidWork = false;
	Context->Coordinator = &WorkCoordinator.Get();
}
```

Worker Context 배열을 생성하고, 워커 스레드 간 작업 분배 및 동기화를 관리하는 WorkCoordinator를 만듭니다. 그 후 Context를 초기화합니다.

```cpp
// Kick workers
for (int32 Idx = 1; Idx < GReachabilityState.GetNumWorkers(); ++Idx)
{
	Tasks::Launch(TEXT("CollectReferences"), [=]() 
		{
			if (FWorkerContext* Context = WorkCoordinator->TryStartWorking(Idx))
			{
				ProcessSync(Processor, *Context);
			}
		});
}

// Start working ourselves
if (FWorkerContext* Context = WorkCoordinator->TryStartWorking(0))
{
	ProcessSync(Processor, *Context);
}
```

메인 스레드(인덱스 0)을 제외한 나머지 워커들에게 Task를 시킵니다. 그 후 메인 스레드도 Task를 합니다. 여기서 Task는 FastReferenceCollector(Processor).ProcessObjectArray(Context)입니다.

```cpp
// Wait until all work is complete. Current thread can steal and complete everything 
// alone if task workers are busy with long-running tasks.
WorkCoordinator->SpinUntilAllStopped();
```

모든 워커가 작업을 완료할 때까지 기다립니다.

```cpp
for (FWorkerContext* Context : Contexts)
{
	// Reset initial object sets so that we don't process again them in the next iteration or when we're done processing references
	Context->ResetInitialObjects();
	Context->InitialNativeReferences = TConstArrayView<UObject**>();

	// Update contexts' suspended state. This is necessary because some context may have their work stolen and never started (resumed) work.
	if (Context->ObjectsToSerialize.HasWork())
	{
		if (!Context->bDidWork)
		{
			if (GReachabilityState.IsTimeLimitExceeded())
			{
				Context->bIsSuspended = true;
			}
			else
			{
				// Rare case where this context's work has been completely dropped because incremental reachability does not support context stealing
				WorkCoordinator->TryStartWorking(Context->GetWorkerIndex());
				ProcessSync(Processor, *Context);
				checkf(!Context->ObjectsToSerialize.HasWork(), TEXT("GC Context %d was processed but it stil has unfinished work"), Context->GetWorkerIndex());
			}
		}
		check(GReachabilityState.IsTimeLimitExceeded() || !Context->ObjectsToSerialize.HasWork());
	}
	else if (!GReachabilityState.IsTimeLimitExceeded()) // !WithTimeLimit
	{
		// This context's work has been stolen by another context so it never had a chance to spin and clear its bIsSuspended flag
		Context->bIsSuspended = false;
	}
}

if (!GReachabilityState.CheckIfAnyContextIsSuspended())
{
	ReleaseAsyncProcessingContexts(InContext, Contexts);		
}
```

Context의 InitialObjects를 비우고, 아직 Task를 시작하지 못 한 Context들에 대해서 TimeLimit 초과 여부(bIsSuspended) 값을 설정합니다. (ProcessObjectArray() 함수 맨 처음에 bDidWork를 true로 합니다.) 모든 Context가 TimeLimit을 초과하지 않았다면, 모든 Async Context들을 Reset합니다.

</div>
</details> 

### ProcessObjectArray()

<details>
<summary> 전체 코드 [ ProcessObjectArray() ] </summary>
<div markdown="1">

```cpp
	void ProcessObjectArray(FWorkerContext& Context)
	{
		Context.bDidWork = true;
		Context.bIsSuspended = false;
		static_assert(!EnumHasAllFlags(Options, EGCOptions::Parallel | EGCOptions::AutogenerateSchemas), "Can't assemble token streams in parallel");
		
		CollectorType Collector(Processor, Context);

		// Either TDirectDispatcher living on the stack or TBatchDispatcher reference owned by Collector
		decltype(GetDispatcher(Collector, Processor, Context)) Dispatcher = GetDispatcher(Collector, Processor, Context);

StoleContext:
		// Process initial references first
		Context.ReferencingObject = FGCObject::GGCObjectReferencer;
		for (UObject** InitialReference : Context.InitialNativeReferences)
		{
			Dispatcher.HandleKillableReference(*InitialReference, EMemberlessId::InitialReference, EOrigin::Other);
		}

		TConstArrayView<UObject*> CurrentObjects = Context.InitialObjects;
		while (true)
		{
			Context.Stats.AddObjects(CurrentObjects.Num());
			ProcessObjects(Dispatcher, CurrentObjects);

			// Free finished work block
			if (CurrentObjects.GetData() != Context.InitialObjects.GetData())
			{
				Context.ObjectsToSerialize.FreeOwningBlock(CurrentObjects.GetData());
			}

			if (Processor.IsTimeLimitExceeded())
			{
				FlushWork(Dispatcher);
				Dispatcher.Suspend();
				SuspendWork(Context);
				return;
			}

			int32 BlockSize = FWorkBlock::ObjectCapacity;
			FWorkBlockifier& RemainingObjects = Context.ObjectsToSerialize;
			FWorkBlock* Block = RemainingObjects.PopFullBlock<Options>();
			if (!Block)
			{
				if constexpr (bIsParallel)
				{
					FSlowARO::ProcessUnbalancedCalls(Context, Collector);
				}

StoleARO:
				FlushWork(Dispatcher);

				if (	 Block = RemainingObjects.PopFullBlock<Options>(); Block);
				else if (Block = RemainingObjects.PopPartialBlock(/* out if successful */ BlockSize); Block);
				else if (bIsParallel) // if constexpr yields MSVC unreferenced label warning
				{
					switch (StealWork(/* in-out */ Context, Collector, /* out */ Block, Options))
					{
						case ELoot::Nothing:	break;				// Done, stop working
						case ELoot::Block:		break;				// Stole full block, process it
						case ELoot::ARO:		goto StoleARO;		// Stole and made ARO calls that feed into Dispatcher queues and RemainingObjects
						case ELoot::Context:	goto StoleContext;	// Stole initial references and initial objects worker that hasn't started working
					}
				}

				if (!Block)
				{
					break;
				}
			}

			CurrentObjects = MakeArrayView(Block->Objects, BlockSize);
		} // while (true)
		
		Processor.LogDetailedStatsSummary();
	}
```

</div>
</details> 

```cpp
Context.bDidWork = true;
Context.bIsSuspended = false;
```

ProcessObjectArray()를 실행한 Context는 bDidWork를 true로 해 주고,  

```cpp
CollectorType Collector(Processor, Context);
```

여기서 CollectorType은 TReachabilityCollector입니다.  

```cpp
// Either TDirectDispatcher living on the stack or TBatchDispatcher reference owned by Collector
decltype(GetDispatcher(Collector, Processor, Context)) Dispatcher = GetDispatcher(Collector, Processor, Context);
```

---

Dispatcher 종류에는 TDirectDispatcher와 TBatchDispatcher가 있는데 GC ReachabilityAnalysis에서는 TBatchDispatcher를 씁니다.  

```cpp
template <EGCOptions Options>
class TReachabilityCollector final : public TReachabilityCollectorBase<Options>
{
	using ProcessorType = TReachabilityProcessor<Options>;
	using DispatcherType = TBatchDispatcher<ProcessorType>;
	using Super =  TReachabilityCollectorBase<Options>;
	using Super::MayKill;

	DispatcherType Dispatcher;

	friend DispatcherType& GetDispatcher(TReachabilityCollector<Options>& Collector, ProcessorType&, FWorkerContext&) { return Collector.Dispatcher; }
```

```cpp
template<class CollectorType, class ProcessorType>
TDirectDispatcher<ProcessorType> GetDispatcher(CollectorType& Collector, ProcessorType& Processor, FWorkerContext& Context)
{
	return { Processor, Context, Collector };
}
```

이 GC코드에서는 TReachabilityCollector, TReachabilityProcessor<Options>이기 때문에, TBatchDispatcher<ProcessorType>가 나옵니다. **decltype를 통해 반환 타입을 결정해 주는 방식**을 썼습니다!  

TDirectDispatcher와 TBatchDispatcher의 차이는,

- TDirectDispatcher : 그때그때 Processor.HandleTokenStreamObjectReference()로 처리
- TBatchDispatcher : Batch에 모아뒀다가 FlushQueues()에서 Processor::HandleBatchedReference()로 한 번에 처리

```cpp
FORCEINLINE static void HandleBatchedReference(FWorkerContext& Context, FResolvedMutableReference Reference, FReferenceMetadata Metadata)
{
	UE::GC::GDetailedStats.IncreaseObjectRefStats(GetObject(Reference));
	if (Metadata.Has(KillFlag))
	{
		checkSlow(Metadata.ObjectItem->GetOwnerIndex() <= 0);
		KillReference(*Reference.Mutable);
	}
	else
	{
		HandleValidReference(Context, Reference, Metadata);
	}
}
```

FastReferenceCollector는 Reference를 빠르게 찾는 역할  
TBatchDispatcher는 Batch에 모아두는 Dispatcher  
TReachabilityProcessor는 Reachability 관련 프로세서  
HandleBatchedReference()는 Batch에 모인 Reference들을 다루는 프로세서의 함수 (아직은 GC에서만 씁니다)  

**이렇게 재사용할 수 있게 각자 기능에 따라 잘 모듈화되어 있는 것이 인상 깊었다. (재사용성을 잘 챙긴 거 같다. 이게 모듈화구나!)**  

---

```cpp
// Process initial references first
Context.ReferencingObject = FGCObject::GGCObjectReferencer;
```

Context의 ReferencingObject 값을 넣어줍니다.

### **FGCObject,** GGCObjectReferencer

<details>
<summary> FGCObject, GGCObjectReferencer </summary>
<div markdown="1">

FGCObject는 생성자에서부터 RegisterGCObject()가 호출됩니다.  
FGCObject의 RegisterGCObject() 사용 시 **Root 객체인 GGCObjectReferencer**에 해당 오브젝트가 추가됩니다. (Object의 EFlags::AddStableNativeReferencesOnly에 따라 true면 InitialReferencedObjects 또는 false면 RemainingReferencedObjects 배열에 추가됩니다.)  

GGCObjectReferencer의 AddReferencedObjects()가 호출되면 InitialReferencedObjects와 RemainingReferencedObjects 배열에 있는 오브젝트들의 AddReferencedObjects()를 호출합니다.  

따라서 FGCObject를 상속 받아서 AddReferencedObjects() 가상함수를 오버라이드해서 UObject* 멤버 변수를 FReferenceCollector에 추가하면 해당 멤버 변수를 GC에 등록할 수 있습니다.  

그래서 UObject가 아닌 **일반 C++ Object**를 사용할 때, FGCObject를 상속 받아서 UObject 포인터인 멤버 변수를 GC에 등록할 수 있습니다.  

소멸자에서 UnregisterGCObject로 GGCObjectReferencer에서 제거합니다.  

따라서 AddToRoot() 대신 사용하면 안전하게 사용할 수 있습니다!  

</div>
</details> 

### TStrongObjectPtr

<details>
<summary> TStrongObjectPtr </summary>
<div markdown="1">

FGCObject와 관련된 TStrongObjectPtr에 대해 적어보면,  
우선, TStrongObjectPtr은 UObject(포인터, 참조)만 보관할 수 있습니다.  
보관하게 되면 해당 UObject를  

```cpp
TUniquePtr< UEStrongObjectPtr_Private::TInternalReferenceCollector<ReferencerNameProvider> > ReferenceCollector;
```

TStrongObjectPtr의 멤버 변수인 TUniquePtr<TInternalReferenceCollector> ReferenceCollector에 대입하고, ReferenceCollector의 멤버변수인 Object에 보관됩니다.  

ReferenceCollector는 **FGCObject**를 상속 받은 클래스이고, AddReferencedObjects()를 멤버변수인 Object를 GC에 등록하도록 오버라이드했습니다.  
즉, TStrongObjectPtr은 UObject를 GC에 등록해 주는 포인터입니다.  

</div>
</details>

```cpp
for (UObject** InitialReference : Context.InitialNativeReferences)
{
	Dispatcher.HandleKillableReference(*InitialReference, EMemberlessId::InitialReference, EOrigin::Other);
}
```

우선, InitialNativeReferences를 처리합니다.  

TBatchDispatcher의 HandleKillableReference()  

```cpp
FORCEINLINE_DEBUGGABLE void HandleKillableReference(UObject*& Object, FMemberId MemberId, EOrigin Origin)
{
	QueueReference(Context.GetReferencingObject(), Object, MemberId, ProcessorType::MayKill(Origin, true));
}
```

GetReferencingObject()는 Context의 ReferencingObject를 가져오는 함수입니다.  
여기서 MayKill()은 EGCOptions::EliminateGarbage가 켜져있으면 true입니다. 

```cpp
FORCEINLINE_DEBUGGABLE void QueueReference(const UObject* ReferencingObject,  UObject*& Object, FMemberId MemberId, EKillable Killable)
{
	if (Killable == EKillable::Yes)
	{
		FPlatformMisc::Prefetch(&Object);
		KillableBatcher.PushReference(FMutableReference{&Object});
	}
	else
	{
		ImmutableBatcher.PushReference(FImmutableReference{Object});
	}
}
```

Killable이 Yes면 Prefetch 후 KillableBatcher에 Push합니다.  

Killable이 No면 ImmutableBatcher에 Push합니다.  

(ReferencingObject은 쓰지 않습니다. )  

---

### FPlatformMisc

FPlatformMisc는 다양한 유틸리티 함수를 Platform마다 다르게 구현할 수 있게 만든 인터페이스입니다.  
FGenericPlatformMisc가 공통 인터페이스이고, 이를 플랫폼별로 상속받아 재정의합니다. (예 : 윈도우는 WindowsPlatformMisc**)**  

실제 코드에서는 typedef를 통해 현재 빌드 타겟 플랫폼에 맞는 클래스로 자동 연결됩니다.  

```cpp
#if WINDOWS_USE_FEATURE_PLATFORMMISC_CLASS
typedef FWindowsPlatformMisc FPlatformMisc;
#endif
```

그래서 개발자는 함수 사용 시 플랫폼별로 다르게 쓸 필요 없이, FPlatformMisc를 쓰면 됩니다.  

---

```cpp
		TConstArrayView<UObject*> CurrentObjects = Context.InitialObjects;
		while (true)
		{
			Context.Stats.AddObjects(CurrentObjects.Num());
			ProcessObjects(Dispatcher, CurrentObjects);

			// Free finished work block
			if (CurrentObjects.GetData() != Context.InitialObjects.GetData())
			{
				Context.ObjectsToSerialize.FreeOwningBlock(CurrentObjects.GetData());
			}

			if (Processor.IsTimeLimitExceeded())
			{
				FlushWork(Dispatcher);
				Dispatcher.Suspend();
				SuspendWork(Context);
				return;
			}

			int32 BlockSize = FWorkBlock::ObjectCapacity;
			FWorkBlockifier& RemainingObjects = Context.ObjectsToSerialize;
			FWorkBlock* Block = RemainingObjects.PopFullBlock<Options>();
			if (!Block)
			{
				if constexpr (bIsParallel)
				{
					FSlowARO::ProcessUnbalancedCalls(Context, Collector);
				}

StoleARO:
				FlushWork(Dispatcher);

				if (	 Block = RemainingObjects.PopFullBlock<Options>(); Block);
				else if (Block = RemainingObjects.PopPartialBlock(/* out if successful */ BlockSize); Block);
				else if (bIsParallel) // if constexpr yields MSVC unreferenced label warning
				{
					switch (StealWork(/* in-out */ Context, Collector, /* out */ Block, Options))
					{
						case ELoot::Nothing:	break;				// Done, stop working
						case ELoot::Block:		break;				// Stole full block, process it
						case ELoot::ARO:		goto StoleARO;		// Stole and made ARO calls that feed into Dispatcher queues and RemainingObjects
						case ELoot::Context:	goto StoleContext;	// Stole initial references and initial objects worker that hasn't started working
					}
				}

				if (!Block)
				{
					break;
				}
			}

			CurrentObjects = MakeArrayView(Block->Objects, BlockSize);
		} // while (true)
		
		Processor.LogDetailedStatsSummary();
```

<2편에서 이어서>