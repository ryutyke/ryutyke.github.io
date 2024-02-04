---
title: "[UnrealEngine] Gameplay Framework"
excerpt: "언리얼 엔진이 게임 제작을 위해 제공하는 프레임워크 개요"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Gameplay, Framework]

permalink: /unrealengine/gameplay-framework/

toc: true
toc_sticky: true

date: 2024-02-04 17:50:00
last_modified_at: 2024-02-04
---
<br>

[https://docs.unrealengine.com/5.3/ko/gameplay-framework-in-unreal-engine/](https://docs.unrealengine.com/5.3/ko/gameplay-framework-in-unreal-engine/)

## 게임플레이 프레임워크

<mark>언리얼 엔진의 게임플레이 프레임워크</mark>는 프로젝트의 빌딩 블록 역할을 하는 <mark>여러 클래스와 컴포넌트를 제공</mark>합니다. 

<br>

### 액터
<mark>레벨에 배치하거나 스폰할 수 있는 오브젝트의 기본 클래스는 액터입니다.</mark> 액터에는 <mark>액터의 이동 방식과 렌더링 방식을 제어하는 데 사용할 수 있는 액터 컴포넌트</mark> 컬렉션이 포함될 수 있습니다. 액터는 플레이 도중 네트워크를 통해 프로퍼티 및 함수 호출의 리플리케이션을 지원합니다. 

<br>

### 카메라
<mark>카메라는</mark> 플레이어가 월드를 보는 방식과 같은 <mark>플레이어의 시점을 나타냅니다.</mark> <mark>플레이어 컨트롤러는 카메라 클래스를 지정하고 플레이어가 월드를 보는 위치와 방향을 계산하는 데 사용되는 카메라 액터를 인스턴스화</mark>합니다.

<br>

### 폰
<mark>폰 클래스는 "플레이어 또는 AI가 제어할 수 있는 액터들"의 베이스 클래스입니다.</mark> 폰은 월드 내에서 플레이어 또는 AI 엔티티의 물리적 표현입니다. <mark>캐릭터는 걸어 다닐 수 있는 특수한 유형의 폰입니다.</mark> 기본적으로 <mark>컨트롤러와 폰은 "일대일 관계"</mark>로, <mark>각 컨트롤러는 주어진 시간에 하나의 폰만 제어할 수 있습니다.</mark>

<br>

### 컨트롤러
<mark>컨트롤러</mark>는 캐릭터처럼 <mark>"폰 또는 폰에서 파생된 클래스(캐릭터)"</mark>를 빙의하여 그 <mark>동작을 제어할 수 있는 비물리적 액터</mark>입니다. <mark>"플레이어 컨트롤러"</mark>는 인간이 폰을 제어하는 데 사용되며, <mark>"AI 컨트롤러"</mark>는 폰의 인공 지능을 구현하는 데 사용됩니다. 컨트롤러는 <mark>Possess 함수</mark>를 사용하여 폰을 제어하고, <mark>UnPossess 함수</mark>를 사용하여 폰의 제어권을 포기할 수 있습니다.

<br>

### 게임플레이 타이머
<mark>게임플레이 타이머</mark>는 <mark>지연 후 또는 일정 시간 동안</mark> 이벤트를 트리거하는 특정 <mark>함수 포인터에</mark> 대한 <mark>비동기 콜백</mark>을 생성합니다.

<br>

### 게임모드
게임 프레임워크의 베이스는 게임모드입니다. <mark>레벨이 게임플레이를 위해 초기화될 때 AgameModeBase 액터가 인스턴스화됩니다.</mark> 게임모드는 <mark>게임의 규칙을 설정하며, 서버에만 인스턴스화되고 클라이언트에는 존재하지 않습니다.</mark>

<br>

### 플러그인
게임 기능 및 모듈형 게임플레이 플러그인은 개발자가 프로젝트에 독립형 기능을 만들 수 있도록 도와줍니다. 이러한 플러그인은 프로젝트의 코드베이스를 깔끔하고 읽기 쉽게 유지하며 관련 없는 기능 간의 우발적인 상호 작용이나 종속성을 방지합니다.

<br>

### 사용자 인터페이스
<mark>사용자 인터페이스(UI)</mark> 및 헤드업 디스플레이(HUD)는 <mark>플레이어에게 게임에 대한 정보를 제공</mark>하고 경우에 따라 플레이어가 게임과 상호 작용할 수 있도록 하는 방식입니다.

 <br>

### 프레임워크 클래스 관계

<div>
    <img src="/assets/images/posts_img/ue_framework.png" alt="thumbnail" width="100%" min-width="700px" itemprop="image">
</div>

<mark>게임은 GameMode(규칙)와 GameState(상태)로 구성</mark>됩니다. 게임에 참여하는 인간 플레이어는 PlayerController입니다. 게임에서 Pawn(Character)에 빙의하여 레벨에 물리적으로 표현할 수 있습니다. PlayerController는 플레이어에게 입력 컨트롤, HUD, PlayerCameraManager를 제공합니다.
