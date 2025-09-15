---
title: "스탯 컴포넌트 구현(스탯, 버프)"
excerpt: "프로젝트 이쩜오"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, stat, component]

permalink: /unrealengine/stat_component/

toc: true
toc_sticky: true

date: 2025-08-27 17:00:00
last_modified_at: 2025-08-27
---
<br>


# StatComponent

팀프로젝트에서 스탯, 버프를 관리하는 스탯 컴포넌트를 만들게 됐습니다.  

코드 (깃허브)  
- [헤더 파일 (DSStatComponent.h)](https://github.com/ryutyke/Project25L/blob/main/Source/Project25L/Components/DSStatComponent.h){:target="_blank"}  
- [소스 파일 (DSStatComponent.cpp)](https://github.com/ryutyke/Project25L/blob/main/Source/Project25L/Components/DSStatComponent.cpp){:target="_blank"}

<br>

### 스탯 초기화
UDSStatComponent::InitializeStats() 함수를 PossessedBy 에서 호출해 서버에서 초기화해 줬습니다.  

DataSubsystem에서 캐릭터 스탯 정보를 가지고 있는 테이블에서 해당 캐릭터에 해당하는 TableRow를 가져와서 초기화했습니다.

서버에서 클라이언트로 복제되는 구조이기에, 서버에서는 OnRep 함수를 직접 호출해 줬습니다.  


### 버프
버프 하나하나를 FBuffEntry로 관리하고, 캐릭터가 받고 있는 버프들을 FBuffArray에서 관리했습니다.

FBuffEntry에는
1. StatType (Enum)
2. OperationType
3. BuffID
4. BuffValue

FBuffArray에는
1. FBuffEntry Array
2. NextBuffID
3. BuffTimerHandles (TMap 버프 ID -> 타이머 핸들)

이렇게 뒀습니다.

버프 추가 시, FBuffEntry를 만들어서 FBuffArray에 넣어주고 RemoveBuff() 함수를 바인드한 타이머를 작동시켰고
RemoveBuff()에서는 BuffID로 해당 버프를 찾아서 제거해 줬습니다.

<br>

#### 버프 적용  

각 스탯에 해당하는 Enum 값인 StatType을 대응되는 Stat 구조체인 FDSCharacterStat의 멤버 변수 포인터와 매핑하는 Map을 사용해서  

```cpp
static const TMap<EDSStatType, float FDSCharacterStat::*> StatMemberMap =
{
	{EDSStatType::MaxHP,          &FDSCharacterStat::MaxHP},
	{EDSStatType::Attack,         &FDSCharacterStat::Attack},
	{EDSStatType::Defense,        &FDSCharacterStat::Defense},
	{EDSStatType::Luck,           &FDSCharacterStat::Luck},
	{EDSStatType::MoveSpeed,      &FDSCharacterStat::MoveSpeed},
	{EDSStatType::AttackSpeed,    &FDSCharacterStat::AttackSpeed},
	{EDSStatType::AttackRange,    &FDSCharacterStat::AttackRange},
	{EDSStatType::HPRegen,        &FDSCharacterStat::HPRegen},
	{EDSStatType::HPSteal,        &FDSCharacterStat::HPSteal},
	{EDSStatType::DamageBoost,    &FDSCharacterStat::DamageBoost},
	{EDSStatType::DamageReduction,&FDSCharacterStat::DamageReduction},
	{EDSStatType::CDReduction,    &FDSCharacterStat::CDReduction},
	{EDSStatType::CritChance,     &FDSCharacterStat::CritChance},
	{EDSStatType::CritDamage,     &FDSCharacterStat::CritDamage},
};
```

각 스탯들을 순회하며,  

```cpp
for (const TPair<EDSStatType, float FDSCharacterStat::*>& Pair : StatMemberMap)
	{
		const EDSStatType& StatType = Pair.Key;
		float FDSCharacterStat::* MemberPtr = Pair.Value;
		CurrentStat.*MemberPtr = GetFinalStat(StatType);
	}
```

버프를 적용한 최종 스탯으로 업데이트 해 주는 방식으로 구현했습니다.  

```cpp
float UDSStatComponent::GetFinalStat(EDSStatType StatType) const
{
	float TotalAddend = 0.f;
	float TotalMultiplier = 0.f;

	for (const FBuffEntry& Entry : ActiveBuffs.Entries)
	{
		if (Entry.StatType == StatType)
		{
			if (Entry.OperationType == EOperationType::Additive)
			{
				TotalAddend += Entry.BuffValue;
			}
			else if (Entry.OperationType == EOperationType::Multiplicative)
			{
				TotalMultiplier += Entry.BuffValue;
			}
			else if (Entry.OperationType == EOperationType::Percent)
			{
				TotalMultiplier += (Entry.BuffValue / 100);
			}
		}
	}

	// 최종 스탯 = (기본 스탯 + Additive 수치) * (1 + Multiplicative 수치)
	float DefaultStatValue = GetDefaultStatByEnum(StatType);
	float FinalStat = (DefaultStatValue + TotalAddend) * (1.f + TotalMultiplier);
	return FinalStat > 0.f ? FinalStat : 0.f;
}
```

<br>

### 네트워크 최적화

엔진 코드를 통해 NetSerialize와 FastArrayDeltaSerialize에 대해 공부해서,  
[https://ryutyke.github.io/unrealengine/fast_array_serialize/](https://ryutyke.github.io/unrealengine/fast_array_serialize/){:target="_blank"}

NetSerializer를 통해 다른 플레이어에게 버프 상태를 UI로 보여주기 위해 **필요한 정보만 직렬화**하게 했고,  

```cpp
bool NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
	{
		Ar << StatType;
		Ar << OperationType;
		return true;
	}
```

FastArrayDeltaSerializer를 통해 MarkArrayDirty로 버프 Array가 변경된 것을 표시하고, 이 경우에만 직렬화하게 했습니다.

```cpp
bool NetDeltaSerialize(FNetDeltaSerializeInfo& DeltaParms)
	{
		return FFastArraySerializer::FastArrayDeltaSerialize<FBuffEntry, FBuffArray>(Entries, DeltaParms, *this);
	}
```

<br>

### Cheat Manager

CheatManager를 사용해서, 특정 기능을 쉽게 테스트할 수 있게 했습니다.

<div>
    <img src="/assets/images/posts_img/statcomponent/image0.png" alt="cheatmanager" width="100%" min-width="700px" itemprop="image">
</div>

