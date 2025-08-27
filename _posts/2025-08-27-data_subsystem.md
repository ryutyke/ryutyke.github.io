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

해당 개발에서는 메모리 최적화보다는 생산성을 위한 사용 편리함을 우선시했습니다. 
개발 초반이라 최적화보다 필요한 기능들을 빠르게 구현하는 것이 우선이기도 했습니다.
나름 잘 사용하긴 했지만 아쉬움이 많은 코드입니다. 그렇지만 좋은 경험이었습니다.


**구현하고, 써 보며 느낀. 다음에 구현할 때 참고할 내용은**
직접 Load 해서 사용하는 함수와 자동으로 Load/Unload 해 주는 기능 둘 다 제공하는 것이 좋을 거 같습니다. 
직접 Load 하는 것은 'PrimaryAsset 클래스를 받아서 PrimaryAsset을 비동기 로딩하고, 로딩된 AssetIds나 로딩된 오브젝트 포인터 배열을 레퍼런스를 콜백하는 함수'가 기본이고, 
자동으로 Load/Unload는 UWorldSubsystem을 사용해서 World별로 시작할 때 원하는 Asset을 Load 해 주고 이전 World 것은 Unload. 중복되는 애는 해제 안 되게. 구조체로 월드 이름이랑 시작 시 로드할 에셋들 보관하면 될 거 같습니다. 
번들도 공부해서 활용해 보는 것이 좋을 것 같습니다. 


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


### 생겼던 문제

이건 나중에 생겼던 문제였는데
원래는 enum에 None 값이 없었고, 그래서 데이터 에셋에도 None 값에 해당하는 데이터 테이블이 없었습니다. 
이 None을 두는 것의 중요성을 깨닫는 일이었습니다.

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image3.png" alt="datatables" width="100%" min-width="100px" itemprop="image">
</div>

캐릭터 타입을 나타내는 Enum 값을 넣으면 그 캐릭터 스킬 관련 데이터 테이블의 Enum 값을 돌려주는 이런 함수가 있습니다.

근데 여기서 실수로 CharacterType이 None이 되는 상황이 발생해 EDataTableType()이 반환되는 일이 있었습니다.
EDataTableType()는 해당 Enum의 첫 번째 값이 반환됩니다.
그래서 엉뚱한 데이터 테이블이 반환되는 문제가 있었습니다.

디버깅을 통해 어디가 문제인지 빠르게 발견할 수 있었지만, None이었다면 더 쉽게 알 수 있었을 것이고 
정말정말 더 중요한 건, **발견하기 어려운 상황이 발생했을지도 모른다는 겁니다.**

그래서 None을 넣었습니다. 

<br>

## Data Asset

Data Asset을 관리하는 것도 Enum을 Key로 사용한 Map 기반으로 만들었습니다. 
생각했던 다른 방법도 있었던 거 같은데, Find가 O(1)이라는 점이 Map을 사용한 가장 큰 이유였던 것 같습니다.
근데 지금 생각해 보면 해당 시스템에서 **Map까지 제공할 필요가 없었을 거 같습니다.**
개발 편의를 우선시 하다보니 그렇게 된 것도 있겠지만, 설계 실력이 안 좋았던 것 같습니다. 

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
전사, 마법사, 궁수 캐릭터가 있고, 초기화를 위한 각 캐릭터별로 데이터 에셋(UDSCharacterDataAsset) 이 있습니다.
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

```cpp KeyType Key = static_cast<KeyType>(AssetPtr->GetKey()); ```



이렇게 데이터 에셋을 보관하는 TMap<KeyType, AssetType*>을 어떻게 관리할까 고민했었습니다.

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image6.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

그러다가 자동으로 만들어지는 정적 Map을 생각하게 됐습니다. TMap이 게임 실행 동안 항상 존재해야 한다는 메모리 측면의 문제보다 개발상 편의를 선택했습니다.
이때 스스로를 칭찬했는데, 지금 생각해 보면 **애초에 이 서브시스템에서 Map까지 제공하게 설계한 것부터 정말 바보였던 거 같습니다.**

```cpp

template<typename KeyType, typename AssetType>
void UDSGameDataSubsystem::LoadDataAssetAsync()
{
	TMap<KeyType, AssetType*>& OutDataAssetMap = GetDataMap<KeyType, AssetType>();

```

Load한 에셋들을 보관할 TMap을 이렇게 자동으로 얻을 수 있고,

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image7.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

편합니다.

<br>

---

참고로, 클래스로 이렇게 타입을 알 수 있습니다.
```cpp 
const FPrimaryAssetType& ThisPrimaryAssetType = AssetType::StaticClass()->GetDefaultObject()->GetPrimaryAssetId().PrimaryAssetType;
```
