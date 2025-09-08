---
title: "데이터 서브시스템 구현"
excerpt: "프로젝트 이쩜오"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Data, Subsystem]

permalink: /unrealengine/data_subsystem/

toc: true
toc_sticky: true

date: 2025-08-27 22:30:00
last_modified_at: 2025-08-27
---
<br>


# DataSubsystem

팀프로젝트에서 데이터 기반 프로그래밍을 하기 위해 데이터 테이블과 데이터 에셋을 관리하는 데이터 서브시스템을 만들게 됐습니다.  

코드 (깃허브) :  
[헤더 파일](https://github.com/ryutyke/Project25L/blob/main/Source/Project25L/System/DSGameDataSubsystem.h)  
[소스 파일](https://github.com/ryutyke/Project25L/blob/main/Source/Project25L/System/DSGameDataSubsystem.cpp)

<br>

## Data Table

```cpp
UFUNCTION(BlueprintCallable, Category = "Data Table")
const UDSDataTables* GetDataTables() const { return DataTables; }
```

데이터 테이블은 하나의 데이터 에셋(UDSDataTables)에 저장해 사용했습니다. (물론, 필요하면 다른 데이터 에셋에 두고 사용해도 됩니다.)  

각 데이터 테이블을 나타내는 Enum을 Key로 사용한 TMap을 통해 필요한 데이터 테이블을 가져다 쓰는 방식입니다. 

```cpp

// DSEnums.h

UENUM(BlueprintType)
enum class EDataTableType : uint8
{
	None,
    CharacterData,
	NonCharacterData,
	GirlSkillAttributeData,
	BoySkillAttributeData,
	MisterSkillAttributeData,
	ItemData,
	InputData,
	ItemVehicleData,
	ItemPotionData,
	ItemGrenadeData,
	ItemAccessoryData,
	DungeonData,
	WeaponData,
};

```

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image1.png" alt="datatables" width="100%" min-width="100px" itemprop="image">
</div>

<br>

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image2.png" alt="datatables" width="100%" min-width="100px" itemprop="image">
</div>

TSoftObjectPtr도 가능하게 해서 원한다면 직접 메모리 관리를 할 수 있습니다. 

<br>

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image8.png" alt="datatables" width="100%" min-width="100px" itemprop="image">
</div>

추가로, 이와 같이 많이 쓰일법한 함수들을 제공했습니다.

<br>

### 생겼던 문제

이건 나중에 생겼던 문제였는데, None을 두는 것의 중요성을 깨닫는 일이었습니다.  

원래는 enum에 None 값이 없었고, 그래서 데이터 에셋에도 None 값에 해당하는 데이터 테이블이 없었습니다.  

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image3.png" alt="datatables" width="100%" min-width="100px" itemprop="image">
</div>

캐릭터 타입을 나타내는 Enum 값을 넣으면 그 캐릭터 스킬 관련 데이터 테이블의 Enum 값을 돌려주는 이런 함수가 있습니다.

근데 여기서 실수로 CharacterType이 None이 되는 상황이 발생해 EDataTableType()이 반환되는 일이 있었습니다.  

EDataTableType()은 해당 Enum의 첫 번째 값이 반환됩니다.  
그래서 엉뚱한 데이터 테이블이 반환되는 문제가 있었습니다.

디버깅을 통해 어디가 문제인지 빠르게 발견할 수 있었지만, None이었다면 더 쉽게 알 수 있었을 것이고  
정말정말 더 중요한 건, **발견하기 어려운 상황이 발생했을지도 모른다는 겁니다.**

그래서 None을 넣었습니다. 

<br>

## Data Asset

Data Asset을 관리하는 것도 Enum을 Key로 사용한 Map 기반으로 만들었습니다.  

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image4.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

우선, 사용할 데이터 에셋 타입을 설정에서 등록해 줘야 합니다. 디렉터리 등을 명시해 최적화가 가능합니다.

<br>

### Load 함수

>지정한 AssetType에 해당하는 DataAsset들을 비동기 로드하여, 정적 TMap에 등록하는 템플릿 함수.

다양한 에셋 타입에 대해 쓸 수 있도록 템플릿 함수를 사용했습니다. 

템플릿 파라미터로
- KeyType : TMap 등록 시 사용될 키의 타입
- AssetType : 로드할 DataAsset 클래스의 타입
을 받습니다.

예시를 들면,
전사, 마법사, 궁수 캐릭터가 있고, 초기화를 위한 각 캐릭터별로 데이터 에셋(UDSCharacterDataAsset)이 있습니다.  

이들은 캐릭터별로 분류가 가능하니 ECharacterType을 KeyType으로 쓰면 되고, 에셋 타입은 UDSCharacterDataAsset입니다.  

이 데이터 에셋들을 비동기 로드해서 TMap<KeyType, AssetType*>에 저장합니다.

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image5.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

Map에 넣을 때 사용할 Key에 해당하는 Enum 값을 얻기 위해 UDSPrimaryDataAsset에는 GetKey() 라는 순수 가상 함수가 있습니다.  

데이터 에셋을 사용할 경우 이를 구현해서 키를 정해줘야 합니다. 

```cpp
uint32 UDSCharacterDataAsset::GetKey()
{
	uint32 EnumAsUint32 = static_cast<uint32>(Type);
	return EnumAsUint32;
}
```

예를 들어, 이렇게 uint32로 Key 값을 반환하게 구현하면, 해당 함수에서 KeyType으로 캐스팅해서 사용합니다.  

`KeyType Key = static_cast<KeyType>(AssetPtr->GetKey());`



이렇게 데이터 에셋을 보관하는 TMap<KeyType, AssetType*>을 어떻게 관리할까 고민했었습니다.

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image6.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

그러다가 자동으로 만들어지는 정적 Map을 생각하게 됐습니다. 

```cpp

template<typename KeyType, typename AssetType>
void UDSGameDataSubsystem::LoadDataAssetAsync()
{
	TMap<KeyType, AssetType*>& OutDataAssetMap = GetDataMap<KeyType, AssetType>();

```

Load한 에셋들을 보관할 TMap을 프로퍼티로 선언하지 않아도, 자동으로 얻을 수 있어서 사용하기에 편했습니다.

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image7.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

<br>

---


## 구현, 사용하고 느낀 점

일단 프로젝트 초반이라 사용에 문제 없을 정도로만 빠르게 구현하고 나중에 보완하자는 마음으로 구현한 부분도 있고, 처음이라 설계를 잘 못한 부분도 있는 것 같습니다.  

아직까지는 게임의 규모가 큰 편이 아니라 문제가 없었지만, 규모가 커지거나 규모가 큰 게임을 만들게 된다면 이런 부분들을 고려해 보면 좋을 것 같습니다.  

1. 메모리 해제 로직  
지금 구현한 내용은 메모리 해제를 구현하지 않았기에 50%만 구현됐다고 할 수 있을 것 같습니다. 사용자에게 모든 메모리 관리 책임을 위임했습니다.  
핸들 등을 관리해서 해제 함수도 제공하고, Deinitialize()에서 자동으로 해제도 해주는 해제 로직을 구현해야 합니다.
또한, 번들 시스템을 공부해서 활용해 보는 것이 좋을 것 같습니다.  

2. 생명 주기 세분화  
여러 생명 주기의 Subsystem을 사용하는 등, 생명 주기를 세분화하면 메모리를 효율적으로 관리할 수 있을 것 같습니다.  
더 생명 주기가 짧은 서브시스템 교체에 더 긴 서브시스템이 개입해서 중복되는 에셋은 Unload/Load 하지 않게 구현하는 것도 고려해 볼 수 있을 것 같습니다.  
  
3. 로드한 에셋 관리  
근데 지금 생각해 보면 **모든 에셋을 Enum을 Key로 사용한 Map으로 관리하는 것**은 좋지 않은 것 같습니다.  
Enum을 사용해야 한다는 제약이 생기는 것도 문제고, 데이터 서브시스템이 로드한 모든 에셋 포인터까지 관리해 줄 필요도 없는 것 같습니다.  

상황에 따라, 
- 다른 값을 Key로 쓰거나,  
- Map을 쓰지 않거나,  
- 에셋 포인터까지 관리해 줄 필요가 없거나,  
하면 좋을 것 같습니다.  