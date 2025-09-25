---
title: "Fast Array Serialize"
excerpt: "네트워크 최적화"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Fast, Array, Serialize]

permalink: /unrealengine/fast_array_serialize/

toc: true
toc_sticky: true

date: 2025-04-26 20:00:00
last_modified_at: 2025-04-26
---
<br>

# FFastArraySerializer

### 원하는 원소만 직렬화

일반적으로 배열의 Replication은 직렬화된 Base State와 Current State의 메모리 값을 비교(memcmp)해서 차이를 확인할 것입니다. 그리고 바뀐 부분이 있다면 배열 전체를 보낼 것입니다.  

**NetDeltaSerialize**는 struct 정보 등을 가지는 FNetDeltaSerializeInfo를 함수 인자로 제공합니다. 이를 사용해서 배열의 Replication 로직을 커스터마이즈할 수 있습니다.  

**FastArrayDeltaSerialize**는 언리얼 엔진이 제공하는 NetDeltaSerialize의 커스텀 형태입니다.  

배열의 변경된 원소를 개발자가 직접 Dirty 처리하게 하여, 메모리 비교 없이 제거/추가/수정된 원소를 알 수 있고, 그 원소들만 전송하여 데이터 전송을 최소화합니다.  


FastArrayDeltaSerialize는 각 Element의 고유한 키 값인 ReplicationID가 ReplicationKey 값과 매핑되어 있습니다.  
Element 값을 추가하거나 변경할 때 MarkItemDirty() 함수를 직접 호출해서 ReplicationKey 값을 증가시킵니다. 이 값을 사용해서 **변경된 Element를 쉽게** 찾을 수 있습니다.  
또한 **배열이 변경됐는지 쉽게** 확인할 수 있는 ArrayReplicationKey도 있습니다. MarkArrayDirty() 호출 시 값이 증가합니다.  
Element 변경 시 직접 호출하는 MarkItemDirty() 안에 MarkArrayDirty()가 있습니다. 만약 ArrayReplicationKey 값이 변경되지 않았다면 Serialize를 하지 않습니다.  

USTRUCT에 `WithNetDeltaSerializer = true`을 하면 NetDeltaSerialize()가 호출됩니다. FFastArraySerializer 구조체의 경우 NetDeltaSerialize에서 FastArrayDeltaSerialize()가 호출하게 구현하면 됩니다.  

```cpp
USTRUCT()
struct FBuffArray : public FFastArraySerializer
{
	// ...

	bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
	{
		return FFastArraySerializer::FastArrayDeltaSerialize<FBuffEntry, FBuffArray>(Entries, DeltaParms, *this);
	}
	
};

////////////////////

template<>
struct TStructOpsTypeTraits<FBuffArray> : public TStructOpsTypeTraitsBase2<FBuffArray>
{
	enum
	{ 
		WithNetDeltaSerializer = true 
	};
};
```

```cpp
// Class.h

virtual bool NetDeltaSerialize(FNetDeltaSerializeInfo & DeltaParms, void *Data) override
{
	if constexpr (TStructOpsTypeTraits<CPPSTRUCT>::WithNetDeltaSerializer)
	{
		return ((CPPSTRUCT*)Data)->NetDeltaSerialize(DeltaParms);
	}
	else
	{
		return false;
	}
}
```

<br>

### 원하는 프로퍼티만 직렬화

StatType, OperationType, BuffID, BuffValue 중 StatType, OperationType만 동기화가 필요하다고 해서 이 두 개만 직렬화해서 더욱 최적화하고 싶었습니다.  

아래와 같은 방법으로 할 수 있었습니다.  

```cpp
USTRUCT(BlueprintType)
struct FBuffEntry : public FFastArraySerializerItem
{
	//...

	bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
	{
		Ar << StatType;
		Ar << OperationType;
		return true;
	}
	
};

///////////////////////

template<>
struct TStructOpsTypeTraits<FBuffEntry> : public TStructOpsTypeTraitsBase2<FBuffEntry>
{
	enum
	{
		WithNetSerializer = true
	};
};
```

```cpp
// Class.h

virtual bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess, void *Data) override
{
	if constexpr (TStructOpsTypeTraits<CPPSTRUCT>::WithNetSerializer)
	{
		return ((CPPSTRUCT*)Data)->NetSerialize(Ar, Map, bOutSuccess);
	}
	else
	{
		return false;
	}
}
```

<br>

결과 :

#### 서버 :

<div>
    <img src="/assets/images/posts_img/fastarrayserialize/image1.png" alt="result" width="75%" min-width="100px" itemprop="image">
</div>

#### 클라 (BuffID, BuffValue는 동기화되지 않음) :

<div>
    <img src="/assets/images/posts_img/fastarrayserialize/image2.png" alt="result" width="75%" min-width="100px" itemprop="image">
</div>

<br>

---

## 직접 코드 뜯어보기

### FFastArraySerializer

Fast TArray Replication할 TArray를 랩핑하는 구조체입니다.  

- TMap<int32, int32> ItemMap : Item의 ReplicationID to Array Index 매핑
- int32 IDCounter : 0으로 초기화, 새로운 Item이 생길 때마다 1씩 증가하며, Item의 ReplicationID 값 줄 때 씁니다.

### FFastArraySerializerItem

Fast TArray Replication할 TArray의 Item의 Base Class  

int32 ReplicationID : Item을 고유하게 식별하는 ID. 한 번 할당되면 변경되지 않습니다.  

int32 ReplicationKey : Item이 수정될 때마다 증가시킨다. Item이 변경되었는지 비교하는 데 사용됩니다.  

int32 MostRecentArrayReplicationKey : 가장 최신 업데이트 했을 때의 FastArrayReplicationKey를 기록하기 위해 클라이언트 측에서만 사용한다. ACK이 누락된 데이터를 동기화하는 데 사용됩니다.  


### FNetFastTArrayBaseState

**State 데이터를 저장**하는 데 사용됩니다. 

int32 ArrayReplicationKey : State의 ArrayReplicationKey  

```cpp
/** Maps an element's Replication ID to Index. */
TMap<int32, int32> IDToCLMap;
```

IDToCLMap : ReplicationID와 로컬에서의 아이템 배열 인덱스를 맵핑  

```cpp
virtual bool IsStateEqual(INetDeltaBaseState* OtherState)
{
	FNetFastTArrayBaseState * Other = static_cast<FNetFastTArrayBaseState*>(OtherState);
	for (auto It = IDToCLMap.CreateIterator(); It; ++It)
	{
		auto Ptr = Other->IDToCLMap.Find(It.Key());
		if (!Ptr || *Ptr != It.Value())
		{
			return false;
		}
	}
	return true;
}
```

IsStateEqual() : 현재 객체의 IDToCLMap에 있는 Key-Value 쌍과 동일한지 비교  

<br>

### FFastArraySerializerHeader

Fast Array를 직렬화할 때 write/read 되는 헤더 데이터를 가지고 있는 구조체  

int32 ArrayReplicationKey;  

int32 BaseReplicationKey; : old state의 ArrayReplicationKey  

int32 NumChanged;  

TArray<int32, TInlineAllocator<8>> DeletedIndices;  

---

### ACK 누락 처리

reliably send가 아닌 경우, 모든 아이템에 대해서 클라이언트가 수신을 누락한 경우를 체크해서 암묵적 삭제 대상으로 처리한다.  

```cpp
// FFastArraySerializer::TFastArraySerializeHelper<Type, SerializerType>::PostReceiveCleanup()

// ---------------------------------------------------------
// Look for implicit deletes that would happen due to Naks
// ---------------------------------------------------------

// If we're sending data completely reliably, there's no need to do this.
if (!Parms.bInternalAck)
{
	for (int32 idx = 0; idx < Items.Num(); ++idx)
	{
		Type& Item = Items[idx];
		if (Item.MostRecentArrayReplicationKey < Header.ArrayReplicationKey && Item.MostRecentArrayReplicationKey > Header.BaseReplicationKey)
		{
			// Make sure this wasn't an explicit delete in this bunch (otherwise we end up deleting an extra element!)
			if (!Header.DeletedIndices.Contains(idx))
			{
				// This will happen in normal conditions in network replays.
				UE_LOG(LogNetFastTArray, Log, TEXT("Adding implicit delete for ElementID: %d. MostRecentArrayReplicationKey: %d. Current Payload: [%d/%d]"),
					Item.ReplicationID, Item.MostRecentArrayReplicationKey, Header.ArrayReplicationKey, Header.BaseReplicationKey);

				Header.DeletedIndices.Add(idx);
			}
		}
	}
}
```

Header.BaseReplicationKey < Item.MostRecentArrayReplicationKey < Header.ArrayReplicationKey  

ArrayReplicationKey : 현재에 해당하는 전송 시퀀스 번호이다. 직렬화가 호출될 때마다 1씩 늘어난다.  

BaseReplicationKey : 클라이언트가 마지막으로 ACK 응답을 보낸, 즉, 성공적으로 수신을 확인한 ArrayReplicationKey 값이다.  

MostRecentArrayReplicationKey : 각 아이템이 추가 또는 변경될 때, 그 때의 ArrayReplicationKey로 갱신된다.  

즉 이 조건은, 서버가 지난 ACK 이후에 새로운 값을 전송 했으나, 그 이후의 새로운 전송 때까지 수신을 확인(ACK)하지 못 했다는 뜻이다. 따라서 누락된 값을 없애서 서버와 동기화해 준다.  

<br>

---

## 언리얼 엔진 코드 주석에 있는 내용들

FTR : Fast TArray Replication  

FTR은 NetDeltaSerialize의 custom 구현. UStruct의 TArray에 적합.  
배열 요소 제거 serialize 최적화, 그리고 클라이언트에게 add, remove 이벤트 제공.  
단점은 dirty mark 사용. 그리고 클라이언트와 서버 간 list의 순서가 동일하게 보장되지는 않는다.  

Step 1 : FFastArraySerializerItem를 상속받은 struct를 만든다.  

```cpp
USTRUCT()
struct FExampleItemEntry : public FFastArraySerializerItem
{
	GENERATED_USTRUCT_BODY()

	// Your data:
	UPROPERTY()
	int32		ExampleIntProperty;	

	UPROPERTY()
	float		ExampleFloatProperty;

	/** 
	 * Optional functions you can implement for client side notification of changes to items; 
	 * Parameter type can match the type passed as the 2nd template parameter in associated call to FastArrayDeltaSerialize
	 * 
	 * NOTE: It is not safe to modify the contents of the array serializer within these functions, nor to rely on the contents of the array 
	 * being entirely up-to-date as these functions are called on items individually as they are updated, and so may be called in the middle of a mass update.
	 */
	void PreReplicatedRemove(const struct FExampleArray& InArraySerializer);
	void PostReplicatedAdd(const struct FExampleArray& InArraySerializer);
	void PostReplicatedChange(const struct FExampleArray& InArraySerializer);

	// Optional: debug string used with LogNetFastTArray logging
	FString GetDebugString();

};
```

Step 2 : FFastArraySerializer를 상속받은 구조체를 만든다. 이 구조체에는 1단계에서 만든 Item(struct)들을 보관하는 TArray<>가 있어야 한다.  
또한, NetDeltaSerialize() 함수를 만든다.  

```cpp
USTRUCT()
struct FExampleArray: public FFastArraySerializer
{
	GENERATED_USTRUCT_BODY()

	UPROPERTY()
	TArray<FExampleItemEntry>	Items;

	bool NetDeltaSerialize(FNetDeltaSerializeInfo & DeltaParms)
	{
	   return FFastArraySerializer::FastArrayDeltaSerialize<FExampleItemEntry, FExampleArray>( Items, DeltaParms, *this );
	}
};
```

Step 3 : 이 구조체를 만든다. Step2에서 만든 struct를 넣어준다.  

```cpp
template<>
struct TStructOpsTypeTraits< FExampleArray > : public TStructOpsTypeTraitsBase2< FExampleArray >
{
       enum 
       {
					WithNetDeltaSerializer = true,
       };
}; 
```

Step 4 :  

- FExampleArray(step2) 타입의 UPROPERTY를 선언한다.
- Array에 있는 Item을 수정할 때, FExampleArray(step2)의 FFastArraySerializer::MarkItemDirty()를 호출해야 한다. 파라미터로 Dirty Mark할 Item 참조를 넘긴다.
- In your classes GetLifetimeReplicatedProps, use DOREPLIFETIME(YourClass, YourArrayStructPropertyName)
- add/deletes/removes시 notify 받기 위해서 FFastArraySerializerItem(step1)에 이 함수들을 추가할 수 있음. (가상 함수 아니고 템플릿 함수임)

```cpp
 void PreReplicatedRemove(const FFastArraySerializer& Serializer)
 void PostReplicatedAdd(const FFastArraySerializer& Serializer)
 void PostReplicatedChange(const FFastArraySerializer& Serializer)
 void PostReplicatedReceive(const FFastArraySerializer::FPostReplicatedReceiveParameters& Parameters)
```

<br>

---

아래 내용도 엔진 코드 주석에 있는 내용 + 직접 코드 좀 뜯어본 내용입니다. 

### UNetDriver::ServerReplicateActors

UWorld::Tick()에서 서버가 클라이언트로 보낼 액터들을 결정하는 UNetDriver::ServerReplicateActors가 실행된다. 

<details>
<summary> 상세 과정 </summary>
<div markdown="1">

펼침 시작

<br>

UNetDriver::SetWorld() ⇒ RegisterTickEvents()
    
```cpp
void UNetDriver::RegisterTickEvents(class UWorld* InWorld)
{
    if (InWorld)
    {
    	TickDispatchDelegateHandle  = InWorld->OnTickDispatch ().AddUObject(this, &UNetDriver::InternalTickDispatch);
    	PostTickDispatchDelegateHandle	= InWorld->OnPostTickDispatch().AddUObject(this, &UNetDriver::PostTickDispatch);
    	TickFlushDelegateHandle     = InWorld->OnTickFlush    ().AddUObject(this, &UNetDriver::InternalTickFlush);
    	PostTickFlushDelegateHandle		= InWorld->OnPostTickFlush	 ().AddUObject(this, &UNetDriver::PostTickFlush);
    }
}
```
    
UWorld::Tick() ⇒ BroadcastTickFlush() ⇒ TickFlushEvent.Broadcast()
    
OnTickFlush() ⇒ ServerReplicateActors()

<br>

펼침 끝

<br>

</div>
</details> 


ServerReplicateActors_BuildConsiderList() 함수로 복제 대상 액터(ConsiderList)를 골라서  

ServerReplicateActors_PrioritizeActors()에서 우선 순위에 따라 액터를 정렬한다. IsActorRelevantToConnection() ⇒ Actor::IsNetRelevantFor()는 관찰자와 관련이 있는지 확인합니다. 이 함수에서는 IsWithinNetRelevancyDistance()로 관찰자와의 거리도 확인합니다.  

```cpp
bool AActor::IsWithinNetRelevancyDistance(const FVector& SrcLocation) const
{
	return FVector::DistSquared(SrcLocation, GetActorLocation()) < NetCullDistanceSquared;
}
```

또한, FActorPriority에서 GetNetPriority()로 네트워크 우선순위를 정합니다.  

```cpp
const float Time = Channel ? (InConnection->Driver->GetElapsedTime() - Channel->LastUpdateTime) : InConnection->Driver->SpawnPrioritySeconds;
```

마지막 복제 이후 경과 시간인 Time에 NetPriority를 곱한 값이 클수록 Priority가 높습니다.  

근데 조건에 따라 Time에 가중치를 곱해줍니다.  

```cpp
if (ViewTarget && (this == ViewTarget || GetInstigator() == ViewTarget))
{
	// If we're the view target or owned by the view target, use a high priority
	Time *= 4.f;
}
```

가중치 값은 ViewTarget이거나 View Target에 소유되면 가장 높고,  

그게 아니라면,  

```cpp
// If this actor has a location, adjust priority based on location
FVector Dir = GetActorLocation() - ViewPos;
float DistSq = Dir.SizeSquared();
```

관찰자와의 거리와 관찰자가 바라보는 방향과 관찰자에서 해당 액터로 향하는 벡터를 내적한  값을 이용합니다.  

```cpp
// Adjust priority based on distance and whether actor is in front of viewer
if ((ViewDir | Dir) < 0.f)
{
	if (DistSq > NEARSIGHTTHRESHOLDSQUARED)
	{
		Time *= 0.2f;
	}
	else if (DistSq > CLOSEPROXIMITYSQUARED)
	{
		Time *= 0.4f;
	}
}
else if ((DistSq < FARSIGHTTHRESHOLDSQUARED) && (FMath::Square(ViewDir | Dir) > 0.5f * DistSq))
{
	// Compute the amount of distance along the ViewDir vector. Dir is not normalized
	// Increase priority if we're being looked directly at
	Time *= 2.f;
}
else if (DistSq > MEDSIGHTTHRESHOLDSQUARED)
{
	Time *= 0.4f;
}
```

앞뒤 여부, 거리, 그리고, FMath::Square(ViewDir | Dir) > 0.5f * DistSq  
이 부분은 Dir가 정규화된 벡터가 아니어서 사실상 (cosθ 제곱 > 0.5)와 같습니다. cosθ > 루트2/2 니깐 즉, θ < 45도. 뷰어가 액터를 45도 이내로 바라보고 있을 때를 뜻합니다.  

ServerReplicateActors_ProcessPrioritizedActorsRange()에서 각각에 대해 UActorChannel을 생성 및 재사용하고, 그 채널의 ReplicateActor 함수를 호출한다.  

ReplicateActor 함수에서 ReplicateProperties()를 호출한다.  
여기서 CompareProperties()로 어떤 프로퍼티가 변경되었는가를 판단한 뒤, FOutBunch라는 패킷 버퍼에 WriteContentBlockPayload() 함수를 호출해 클라이언트로 전송할 데이터를 준비한다.  

<br>

### 프로퍼티 비교

UActorChannel에는 프로퍼티 비교를 위한 두 가지 방법이 있다.  

#### **Recent 버퍼 (고정 크기 버퍼)**

TArray<uint8> 타입의 평탄화(1차원화)된 바이트 블록이다. 이는 클라이언트가 마지막으로 받은 액터의 프로퍼티 값을 그대로 저장한다. 서버는 이 버퍼와 현재 액터의 값을 비교해서 무엇이 바뀌었는지 판단한다.  

int, float, pointer 등 크기가 고정(atomic)인 타입은 잘 처리하지만, 그러나 TArray처럼 내부에 포인터와 크기 정보를 함께 가지는 함께 가지는 동적(dynamic) 타입은 처리하지 못 한다. 이러한 데이터는 평탄화된 블록에 맞지 않다. 이를 해결하기 위해 RecentDynamicState가 있다.  

#### **RecentDynamicState 맵 (동적 state용 맵)**

이 Map을 통해 프로퍼티의 RepIndex를 통해 동적 프로퍼티의 base state를 얻을 수 있다.  

<br>

### 직렬화 방식 NetSerialize & NetDeltaSerialize

#### **NetSerialize**

Recent 버퍼로 관리할 수 있는 프로퍼티는 모두 NetSerialize로 직렬화가 가능하다. NetSerialize는 단순히 FArchive에 읽기/쓰기 (read/write)만 한다. Recent 버퍼 값을 비교해 쉽게 변경 여부를 확인 가능하고, 바뀐 데이터만 전송할 수 있다.  

#### **NetDeltaSerialize**

동적 프로퍼티는 NetDeltaSerialize로만 직렬화가 가능하다. 이전에 저장해 둔 “Base State”를 사용해서, 클라이언트에게 보낼 변경분인 “Delta State”와 델타 비교를 위한 다음 Base State가 될 “Full State”를 만든다. 직렬화뿐만 아니라 “어떤 부분이 바뀌었는지” 확인하는 Diffing을 해야 한다.  

<br>

### Base States and dynamic properties replication.

리플리케이션 시스템과 UActorChannel에서 Base State는 어떤 것이든 될 수 있으며, 오직 INetDeltaBaseState* 타입으로만 다룬다.  
UActorChannel::ReplicateActor는 각 프로퍼티에 대해 FProperty::NetSerializeItem을 호출할지, 아니면 FProperty::NetDeltaSerializeItem을 호출할지 최종적으로 결정한다.  
INetDeltaBaseState 객체들은 NetDeltaSerialize 함수 내에서 생성되므로, 리플리케이션 시스템이나 UActorChannel 에서는 그 세부 구현을 알 필요가 없다.  
Delta Serialization 방법에는 Generic Replication과 Fast Array Replication, 이렇게 두 가지가 있다.  

1. **Generic Delta Replication**

FArrayProperty와 FStructProperty를 직렬화하는 디폴트 방법이다. 배열과 구조체의 하위 프로퍼티 중 NetSerialize가 가능한 프로퍼티에 대해서 동작한다.  

아래 세 곳에서 구현된다.(근데 엔진 코드 찾아봤을 때는 없었다!)  

- FProperty::NetDeltaSerializeItem
- FStructProperty::NetDeltaSerializeItem
- FArrayProperty::NetDeltaSerializeItem

과정  

1. 현재 State를 NetSerialize로 직렬화해서 “Full State”에 저장한다.
2. `memcmp`로 이전 Base State와 비교해 바뀌었는지 확인한다. 바뀐 경우, 비교 방법이 구현된 FProperty는 “Diff State”를 기록한다.
3. FStructProperty와 FArrayProperty는 필드나 배열 요소를 순회하며, FProperty의 함수를 호출하는 방식을 사용한다. (이들은 meta data를 넘겨준다.)

#### **Custom Net Delta Serialization**

구조체에 WithNetDeltaSerializer trait을 가지고 있는 경우, Generic Delta Replication가 아니라 FStructProperty::NetDeltaSerializeItem를 호출한다.  

2. **Fast TArray Replication**

Custom Net Delta Serialization으로 구현된다.  

**State를** 평탄화된 TArray로 표현하는 대신에, **IDs와 ReplicationKeys를 사용한 TMap으로 표현**한다.  
`FFastArraySerializerItem`에 ReplicationID와 ReplicationKey 필드가 정의되어 있다. ReplicationID로 Array에 있는 Item의 ReplicationKey를 매핑한다.  
MarkItemDirty으로 Item을 Dirty 마크할 때, 아이템에게 새로운 ReplicationKey가 주어진다.  

아이템의 ReplicationID는 서버와 클라이언트 사이에 복제되고 동기화된다. 아이템의 배열 인덱스는 복제 및 동기화되지 않는다.  

**Server**

서버에서의 직렬화(쓰기)에서는 아이템 배열의 이전 base state map과 현재 state를 비교한다.  
사라진 아이템은 bunch에서 제거하고, 아이템이 새로 추가되거나 수정된 경우, NetSerialize을 호출한다.  

**Client**

클라이언트에서의 직렬화(읽기)에서는 변경된 아이템 개수와 삭제된 아이템 개수를 읽는다. 그리고 ReplicationID와 로컬에서의 아이템 배열 인덱스를 맵핑한다. 만약 직렬화가 된 적 없는 아이템 ID라면, 필요한 작업을 수행한다.(create, current state에 serialize, delete)  

inner struct의 Delta Serialization 여부 기본 값은 Enabled이다. 구조체 속 변경된 프로퍼티를 추척해서 그 프로퍼티만 전송하는 것이다. 필요 시, 생성자에서 FFastArraySerializer::SetDeltaSerializationEnabled(false)를 호출해서 끌 수 있다. 전역 콘솔 변수 net.SupportFastArrayDelta = 0 으로 완전히 끌 수도 있다.  

<br>