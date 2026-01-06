---
title: "RemoveAtSwap"
excerpt: "배열의 원소 삭제도 O(1)에 할 수 있다."

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, RemoveAtSwap]

permalink: /unrealengine/RemoveAtSwap/

toc: true
toc_sticky: true

date: 2026-01-06 20:00:00
last_modified_at: 2025-01-06
---
<br>

### RemoveAtSwap

일반적으로, 배열의 삭제는 O(n)의 시간이 걸린다. 왜냐하면, 삭제된 원소 뒤에 있는 모든 원소들을 앞으로 한 칸씩 당겨야 하기 때문이다.  

하지만, **순서가 중요하지 않은 경우**에는, 삭제할 인덱스의 원소를 배열의 마지막 원소와 바꿈으로써 O(1)의 시간에 삭제를 수행할 수 있다. 

C++ std::vector에는 이러한 기능이 없지만, Unreal Engine의 TArray에는 `RemoveAtSwap`이라는 메서드가 제공된다.

```cpp
private:
	void RemoveAtSwapImpl(SizeType Index)
	{
		ElementType* Data = GetData();
		ElementType* Dest = Data + Index;

		DestructItem(Dest);

		// Replace the elements in the hole created by the removal with elements from the end of the array, so the range of indices used by the array is contiguous.
		const SizeType NumElementsAfterHole = (ArrayNum - Index) - 1;
		const SizeType NumElementsToMoveIntoHole = FPlatformMath::Min(1, NumElementsAfterHole);
		if (NumElementsToMoveIntoHole)
		{
			RelocateConstructItems<ElementType>((void*)Dest, Data + (ArrayNum - NumElementsToMoveIntoHole), NumElementsToMoveIntoHole);
		}
		--ArrayNum;

		SlackTrackerNumChanged();
	}
public:
	/**
	 * Removes an element at the given location, optionally shrinking the array.
	 *
	 * This version is much more efficient than RemoveAt (O(Count) instead of
	 * O(ArrayNum)), but does not preserve the order.
	 *
	 * @param Index Location in array of the element to remove.
	 * @param AllowShrinking (Optional) By default, arrays with large amounts of slack will automatically shrink.
	 *                       Use FNonshrinkingAllocator or pass EAllowShrinking::No to prevent this.
	 */
	UE_FORCEINLINE_HINT void RemoveAtSwap(SizeType Index, EAllowShrinking AllowShrinking = UE::Core::Private::AllowShrinkingByDefault<AllocatorType>())
	{
		RangeCheck(Index);
		RemoveAtSwapImpl(Index);
		if (AllowShrinking == EAllowShrinking::Yes)
		{
			UE::Core::Private::ReallocShrink<UE::Core::Private::GetAllocatorFlags<AllocatorType>()>(
				sizeof(ElementType),
				alignof(ElementType),
				AllocatorInstance,
				ArrayNum,
				ArrayMax
			);
		}
	}
```

---

<br>

std::vector에서는 RemoveAtSwap을 다음과 같이 구현해서 사용할 수 있을 것이다.

```cpp
template<typename T>
void RemoveAtSwap(std::vector<T>& vec, size_t index)
{
	if (index < vec.size())
	{
		vec[index] = vec.back();
		vec.pop_back();
	}
}
```
