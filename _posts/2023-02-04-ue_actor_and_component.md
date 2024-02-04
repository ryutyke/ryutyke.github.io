---
title: "[UnrealEngine] Actor and Component"
excerpt: "언리얼 엔진이 게임 제작을 위해 제공하는 프레임워크"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Gameplay, Framework, Actor, Component]

permalink: /unrealengine/actor_and_component/

toc: true
toc_sticky: true

date: 2024-02-04 18:05:00
last_modified_at: 2024-02-04
---
<br>

## 액터
- 레벨에 배치할 수 있는 오브젝트.
- Tick() 함수를 통해 틱 가능.
- 게임플레이 코드를 통해 생성 및 소멸 가능.
- C++에서 AActor는 모든 액터의 베이스 클래스.
- 위치와 회전, 스케일 같은 트랜스폼 데이터를 직접 저장하지는 않고, 액터의 루트 컴포넌트에 대한 트랜스폼 데이터를 사용.
- 액터 생성은 SpawnActor()로 할 수 있고, Destroy()로 소멸 가능하다.

액터 종류 예 : StaticMeshActor, CameraActor, PlayerStartActor

<br>

### 상속 계층구조
UObjectBase<br>
UObjectBaseUtility<br>
UObject<br>
AActor<br>
APawn<br>
ACharacter<br>

<br>

- APawn 클래스는 "플레이어 또는 AI가 제어할 수 있는 액터들"의 베이스 클래스입니다.
- ACharacter 클래스는 APawn을 좀 더 특화시킨. 즉, 좀 더 특징을 구체적으로 만든 느낌. SkeletalMeshComponent (Primitive), CapsuleComponent (Primitive), CharacterMovementComponent (Actor)를 Pawn에 추가.  

<br>

## 컴포넌트
액터는 컴포넌트를 담는 컨테이너로 생각할 수 있다.

<br>

### 주요 컴포넌트 타입
- UActorComponent : 베이스 컴포넌트로 입력(Input) 해석과 같은 개념적 기능에 사용된다. TickComponent() 함수를 사용하여 컴포넌트를 틱 시킬 수 있다. 
- USceneComponent : <mark>트랜스폼이 있는 "액터 컴포넌트"</mark>. 액터의 위치와 회전, 스케일은 계층구조의 루트에 있는 씬 컴포넌트에서 가져온다. 액터 컴포넌트에 어태치된 다른 액터 컴포넌트는 어태치된 대상 컴포넌트를 기준으로 한 트랜스폼 정보를 가진다.
- UPrimitiveComponent : 메시나 파티클 시스템 같은 일종의 <mark>그래픽 표현이 있는 "씬 컴포넌트"</mark>. <mark>피직스 및 콜리전 세팅이 있다</mark>. 예시로는,
1. 박스 컴포넌트(Box Component)
2. 캡슐 컴포넌트(Capsule Component)
3. 스태틱 메시 컴포넌트(Static Mesh Component)
4. 스켈레탈 메시 컴포넌트(Skeletal Mesh Component)<br>
이 있다.

<br>

### 상속 계층구조
UObjectBase<br>
UObjectBaseUtility<br>
UObject<br>
UActorComponent<br>
USceneComponent<br>
UPrimitiveComponent<br>
UMeshComponent<br>
UStaticMeshComponent<br>

<br>

### TMI Zone
<mark>ActorComponent는</mark> TickComponent()를 통해 틱 할 수 있는데, <mark>기본적으로는 Tick하지 않게 설정되어 있음.</mark> 그래서 Tick 하려면 <mark>생성자에서 PrimaryComponentTick.bCanEverTick 을 true로 세팅</mark>하고, 생성자 또는 다른 위치에서 <mark>PrimaryComponentTick.SetTickFunctionEnable(true) 을 호출하여</mark> 틱을 실행할 수 있다. PrimaryComponentTick.SetTickFunctionEnable(false)로 틱 실행을 멈출 수도 있다. 

<br>

[https://docs.unrealengine.com/5.3/ko/actors-in-unreal-engine/](https://docs.unrealengine.com/5.3/ko/actors-in-unreal-engine/)
[https://docs.unrealengine.com/5.3/ko/components-in-unreal-engine/](https://docs.unrealengine.com/5.3/ko/components-in-unreal-engine/)