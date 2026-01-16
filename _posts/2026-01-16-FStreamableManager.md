---
title: "FStreamableManager"
excerpt: "FStreamableManager, FStreamableHandle, FStreamable"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, FStreamableManager]

permalink: /unrealengine/FStreamableManager/

toc: true
toc_sticky: true

date: 2026-01-16 20:00:00
last_modified_at: 2025-01-16
---
<br>

## FStreamableHandle

FStreamableManager::RequestAsyncLoad에서 MakeShared를 통해 새 FStreamableHandle가 만들어진다.

FStreamableManager::RequestAsyncLoad 함수 매개변수인 FStreamableAsyncLoadParams에 TargetsToStream(FSoftObjectPath), OnComplete, OnCancel, OnUpdate 델리게이트, 우선순위 등을 넣을 수 있다. 이러한 내용들이 FStreamableHandle로 Move된다.

이제 이 핸들을 사용해서 StartHandleRequests가 시작된다. 이 안에서 StreamInternal()이 호출된다.

## FStreamable

- 코드
<details>
<summary> 전체 코드 [ FStreamable ] </summary>
<div markdown="1">
    ```cpp
    /** Internal object, one of these per object paths managed by this system */
    struct FStreamable
    {
    	/** Hard GC pointer to object */
    	TObjectPtr<UObject> Target = nullptr;
    
    	/** Live handles keeping this alive. Handles may be duplicated. */
    	TArray<FStreamableHandle*> ActiveHandles;
    
    	/** Handles waiting for this to load. Handles may be duplicated with redirectors. */
    	TArray< TSharedRef< FStreamableHandle> > LoadingHandles;
    
    	/** Id for async load request with async loading system */
    	int32 RequestId = INDEX_NONE;
    
    	/** If this object is currently being loaded */
    	bool bAsyncLoadRequestOutstanding = false;
    
    	/** If this object failed to load, don't try again */
    	bool bLoadFailed = false;
    
    	void FreeHandles()
    	{
    		// Clear the loading handles 
    		for (TSharedRef<FStreamableHandle>& LoadingHandle : LoadingHandles)
    		{
    			LoadingHandle->StreamablesLoading--;
    		}
    		LoadingHandles.Empty();
    
    		// Cancel active handles, this list includes the loading handles
    		while (ActiveHandles.Num() > 0)
    		{
    			FStreamableHandle* ActiveHandle = ActiveHandles.Pop(EAllowShrinking::No);
    			if (!ActiveHandle->bCanceled)
    			{
    				// Full cancel isn't safe any more
    
    				ActiveHandle->bCanceled = true;
    				ActiveHandle->OwningManager = nullptr;
    
    				if (ActiveHandle->bReleased)
    				{
    					ActiveHandle->NotifyParentsOfCancellation();
    				}
    				else
    				{
    					// Keep handle alive to stop the cancel callback from dropping the last reference
    					TSharedPtr<FStreamableHandle> SharedHandle = ActiveHandle->AsShared();
    
    					FStreamableHandle::ExecuteDelegate(MoveTemp(ActiveHandle->CancelDelegate), SharedHandle);
    					ActiveHandle->UnbindDelegates();
    					ActiveHandle->NotifyParentsOfCancellation();
    				}
    			}
    		}
    	}
    
    	void AddLoadingRequest(TSharedRef<FStreamableHandle>& NewRequest)
    	{
    		// With redirectors we can end up adding the same handle multiple times, this is unusual but supported
    		ActiveHandles.Add(&NewRequest.Get());
    
    		LoadingHandles.Add(NewRequest);
    		NewRequest->StreamablesLoading++;
    	}
    };
    ```

<br>

</div>
</details> 

최소 로딩 에셋 단위.

에셋 오브젝트를 TObjectPtr로 가르키고 있다.

자신을 로딩한 Streamable Handle을 ActiveHandles에 보관

## FStreamableManager

로드 요청한 FStreamable들을 StreamableItems에 가지고 있다.

```cpp
/** Map of paths to streamable objects, this will be the post-redirector name */
typedef TMap<FSoftObjectPath, struct FStreamable*> TStreamableMap;
TStreamableMap StreamableItems;
```

FStreamable은 로드가 완료됐을 수도, 아직 미완료일 수도 있다.

FStreamableManager::StreamInternal는 로드할 FSoftObjectPath를 주면 FStreamable을 주는 함수이다. 함수 코드를 보면,

1. StreamableItems에 있는지 먼저 확인을 한다. 
2. 있다면, bAsyncLoadRequestOutstanding(미완료)인지 Existing->Target (완료해서 결과물을 Target에 보관)인지 확인한다. 이미 로드 완료 상태면 반환한다.
3. 없다면, StreamableItems에 등록을 한다.
4. bAsyncLoadRequestOutstanding 상태가 아니라면, 다시 말해서 3번에서 등록한 경우, FindInMemory()함수를 통해 메모리에서 로드한다.
5. 만약 로드 성공하면, Existing->Target이 유효해진다. 근데, 만약 EInternalObjectFlags_AsyncLoading이 활성화되어 있다면 bAsyncLoadRequestOutstanding = true 상태가 되고, Target은 로딩이 될 때까지 유효하지 않다(nullptr). 그리고 비동기 로딩을 시작한다.
6. 비동기 로딩인 경우 언제 Target이 유효해지는가? 로딩 완료 콜백 함수로 AsyncLoadCallback()를 넣는다. 이 안에서 FindInMemory()를 호출해서 로딩된 오브젝트를 Target에 넣어준다.
7. 이렇게 로딩이 끝나고 나면 CheckCompletedRequests() 함수가 호출된다. Handle->CompleteLoad()가 호출된다. 여기서 델리게이트를 실행한다. 또한, 만약 Handle->bReleaseWhenLoaded을 설정해뒀다면 핸들이 릴리즈된다. 핸들 릴리즈란 FStreamableManager->ManagedActiveHandles 배열에서 제거되는 것이다.

### 핸들이 제거되면 로드된 오브젝트는 GC되는가?

핸들 소멸자에 그런 내용은 없다. 

FStreamable에서 가리키는 로딩한 오브젝트 TObjectPtr을 GC Reachable 등록하는 부분

```cpp
void FStreamableManager::AddReferencedObjects(FReferenceCollector& Collector)
{
	// If there are active streamable handles in the editor, this will cause the user to Force Delete, which is irritating but necessary because weak pointers cannot be used here
	for (auto& Pair : StreamableItems)
	{
		Collector.AddStableReference(&Pair.Value->Target);
	}
	
	for (TPair<FSoftObjectPath, FRedirectedPath>& Pair : StreamableRedirects)
	{
		Collector.AddStableReference(&Pair.Value.LoadedRedirector);
	}
}
```

FStreamableManager 소멸자에서는 모든 로딩한 오브젝트를 delete한다.

```cpp
FStreamableManager::~FStreamableManager()
{
	FCoreUObjectDelegates::GetPreGarbageCollectDelegate().RemoveAll(this);

	for (const TPair<FSoftObjectPath, FStreamable*>& Pair : StreamableItems)
	{
		Pair.Value->FreeHandles();
	}
	
	TStreamableMap TempStreamables;
	Swap(TempStreamables, StreamableItems);
	for (const TPair<FSoftObjectPath, FStreamable*>& TempPair : TempStreamables)
	{
		delete TempPair.Value;
	}
}
```

그렇다면 핸들 유지 여부랑 상관없이 FStreamableManager가 살아있는 한, 로딩한 오브젝트들은 계속 살아있는가?

**아니다! (중요!!!!!!!!!!!)**

이 부분에서 ActiveHandles이 비어있는지 확인하고, 비어있다면 FStreamable를 제거해버린다.

```cpp
void FStreamableManager::OnPreGarbageCollect()
{
	TRACE_CPUPROFILER_EVENT_SCOPE(FStreamableManager::OnPreGarbageCollect);

	TSet<FSoftObjectPath, DefaultKeyFuncs<FSoftObjectPath>, TInlineSetAllocator<256>> RedirectsToRemove;

	const bool bRemoveRedirects = !StreamableRedirects.IsEmpty();

	// Delete and remove inactive streamables
	for (TStreamableMap::TIterator It(StreamableItems); It; ++It)
	{
		FStreamable* Value = It.Value();
		if (Value->ActiveHandles.IsEmpty())
		{
			if (bRemoveRedirects)
			{
				RedirectsToRemove.Add(It.Key());
			}

			Value->FreeHandles();
			delete Value;
			It.RemoveCurrent();
		}
	}
```

즉, 구조를 정리하면

FStreamableManager에서 로드 요청한 것들이 TMap<FSoftObjectPath, struct FStreamable*> StreamableItems에 등록되며, FStreamable→Target에 로딩된 오브젝트가 저장된다. FStreamableManager는 이러한 Target들을 모두 GC Reachable하게 등록한다. PreGarbageCollect에 FStreamable들의 ActiveHandles를 검사해서 살아있는 핸들이 없다면 FStreamable을 delete한다.

---

그러면 핸들이 bReleaseWhenLoaded 상태라, 로드하자마자 릴리즈되어 버리면 로딩된 오브젝트도 제거되는 것인가?

FStreamableManager에서 다음 GC때 FStreamable를 delete할 것이다. 그러면 FStreamable 속 Target은 이제 GC Reachable 등록이 되지 않은 상태가 된다.

그전에 Target을 따로 보관 및 GC Reachable 등록해주면 된다.

ReleaseHandle보다 CompleteLoad가 먼저 호출되므로, 여기서 로드된 Target을 가져오면 될 것이다.