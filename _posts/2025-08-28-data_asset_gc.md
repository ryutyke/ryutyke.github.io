---
title: "로드한 에셋을 GC에 Reachable 등록을 해 줘야 하는가?"
excerpt: "엔진 코드로 검증"

categories:
  - UnrealEngine
  - pfselect
tags:
  - [UnrealEngine, DataAsset, GC]

permalink: /unrealengine/data_asset_gc/

toc: true
toc_sticky: true

date: 2025-08-28 02:00:00
last_modified_at: 2025-08-28
---
<br>


# 로드한 에셋을 GC에 Reachable 등록을 해 줘야 하는가?

데이터 서브시스템을 구현하다가 궁금해졌던 내용이고, 엔진 코드를 읽어 알아냈습니다.  
(이전에 엔진 GC 코드를 공부한 경험이 있어서, 쉽게 알아낼 수 있었습니다. 약간의 스포가 되겠지만, FStreamableManager가 FGCObject를 상속 받는 것을 보고 AddReferencedObjects 함수를 들어갔습니다.)

해당 게시글입니다. 밑에 나올 TStrongObjectPtr에 대해서도 알 수 있습니다.
[https://ryutyke.github.io/unrealengine/garbage_collection/](https://ryutyke.github.io/unrealengine/garbage_collection/){:target="_blank"}

## 상황 설명

일단 상황을 쉽게 설명드리면,  
(앞선 게시물을 보면 구조를 더 잘 이해할 수 있습니다.)
[https://ryutyke.github.io/unrealengine/data_subsystem/](https://ryutyke.github.io/unrealengine/data_subsystem/){:target="_blank"}

데이터 테이블을 보관하는 데이터 에셋은 TSoftObjectPtr을 LoadSynchronous()로 로드합니다.

```cpp
DataTables = DataTablesAsset.LoadSynchronous();
```

나머지 데이터 에셋은 AssetManager.LoadPrimaryAssets()로 로드해서

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image6.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

정적 맵의 Value에 저장합니다.

---

## 문제

일단 데이터 에셋은 UObject 기반이므로 GC를 신경 써야 합니다. 

DataTables는 UPROPERTY로 가지고 있을 것이어서 걱정이 없었습니다. 

그러나 함수에서 정적 맵을 만들고, 가져오는 방식에서는 TMap을 UPROPERTY로 관리할 수 없었습니다. 

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image6.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image9.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

처음에는 엔진 코드를 뜯어보는 것을 다음으로 미루고, 일단 TStrongObjectPtr을 사용해서 GC에 넘겨줬습니다.


## 해결

### AssetManager.LoadPrimaryAssets()

AssetManager.LoadPrimaryAssets()부터 보시면 

UAssetManager로 Load된 Primary Asset은 FStreamableManager를 통해 Load 됩니다.
<div>
    <img src="/assets/images/posts_img/DataSubsystem/image11.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

Load된 에셋은 FStreamable 객체의 Target 변수에 보관되고 이 FStreamable 객체들은 FStreamableManager에서 Map으로 저장됩니다.
<div>
    <img src="/assets/images/posts_img/DataSubsystem/image12.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image13.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>


그리고 FStreamableManager는 FGCObject를 상속받아 AddReferencedObjects 함수를 오버라이드해 이것들을 GC 참조 추적에 포함시킵니다.
<div>
    <img src="/assets/images/posts_img/DataSubsystem/image14.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/DataSubsystem/image15.png" alt="dataasset" width="100%" min-width="100px" itemprop="image">
</div>

따라서 TStrongObjectPtr을 사용하지 않아도 됩니다.

그러나, 댕글링 포인터 문제가 있습니다.

만약 AssetManager가 관리하는 에셋이 Unload되면, Key에 해당하는 포인터는 메모리 주소를 가리키지만 해당 메모리는 이미 해제되어 댕글링 포인터가 됩니다. 
이는 TWeakObjectPtr을 사용해서 해결할 수 있습니다.

<br>

### SoftObjectPtr.LoadSynchronous()

AssetManager나 FStreamableManager를 쓰지 않고 오직 단순히 직접 Load 한다고 생각하면 됩니다.

따라서 SoftObjectPtr을 LoadSynchronous()해서 얻은 UObject는 따로 관리되지 않습니다.

필요 시, 직접 GC Reachable하게 처리해 줘야 합니다.


