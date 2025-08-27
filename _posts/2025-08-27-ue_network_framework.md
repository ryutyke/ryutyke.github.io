---
title: "언리얼엔진 네트워크 기초 개념"
excerpt: "리슨 서버 공부"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Network]

permalink: /unrealengine/network_listen/

toc: true
toc_sticky: true

date: 2025-08-27 20:30:00
last_modified_at: 2025-08-27
---
<br>

## 언리얼엔진 네트워크 기초

### 서버 (리슨)

- 처음엔 스탠드얼론
- 자기 자신이 서버니깐 prelogin 하지 않는다
- login에서 플레이어 컨트롤러 생성, PostInitializeComponents 등
- login 끝나면 `Listen()`을 통해 넷드라이버 생성을 해서 서버로 전환.
- 서버가 되고부터는 게임모드가 StartPlay 가능, 게임스테이트가 HandleBeginPlay로 BeginPlay 브로드캐스트

### 클라이언트

- 연결 요청하면 서버에 있는 게임 모드에서 로그인 처리. Login에서 클라의 컨트롤러 생성하고 이미 StartPlay를 했으면 BeginPlay까지 함.
- 클라이언트는 서버한테 Network 통해서 초기 값들(컨트롤러) 전송 받아서 초기화 함(NetInit). 이때 PostInitializeComponents도 함
- NetInit이 끝나면, 즉 PostNetInit 까지 끝나면. 그 이후 로직. StartPlay 한 상태면 클라도 BeginPlay 하고 등등

### NetMode, NetDriver

`InternalGetNetMode()`를 통해 NetMode 확인이 가능하다

NetDriver가 가지고 있는
ServerConnection은 클라이언트가 가지고 있는 연결된 커넥션에 대한 정보. 서버에 대한 정보가 아니라 커넥션에 대한 정보이다. 오직 1개이다.
ClientConnection은 서버가 가지고 있는 연결된 커넥션에 대한 정보. 여러 개 가능.

### 용어

커넥션 : 모든 데이터를 전달하는 네트워크 통로

채널 : 구분된 데이터를 전달하는 논리적인 통로. 커넥션과 채널은  1 : N

번치 : 하나의 명령에 대응하는 데이터 묶음.

패킷 : 네트워크를 통해 전달되는 단위 데이터. 숫자 혹은 문자로 구성.

### 커넥션 소유

어떤 액터가 통신을 하기 위해서는 (자신을 소유한) 액터가 커넥션을 소유하고 있어야 함. (커넥션이 액터를 소유함도 맞는 말)

데이터 통신을 관리하기 위한 대표 액터로 플레이어 컨트롤러가 주로 사용됨.

**커넥션을 담당하는 대표 액터는 커넥션에 대한 오너십**을 가진다고 표현한다.

“액터가 커넥션에 소유된다” ⇒ 해당 커넥션이 해당 액터의 네트워크 통로이다.

어떤 액터의 가장 바깥쪽 오너가 커넥션을 담당하는 대표 액터인 경우, 그 액터도 같은 커넥션에 소유된 것이다.

컴포넌트는 컴포넌트의 가장 바깥쪽 액터를 찾고, 그 액터가 어떤 커넥션에 있는지 확인해서 그 커넥션에 소유된다.

`GetNetConnection()`으로 자신이 소유된 커넥션을 가져올 수 있다. 없으면 nullptr 반환.

### 빙의

서버에서는 PostLogin 때 빙의가 일어난다. (Controller의 OnPossess 다음에 Player의 PossessedBy)

클라이언트에서는 OnPossess 함수가 호출되지 않는다. 그러면 오너는 어떻게 정해지는가? 정답은 서버로부터 바뀐 Owner 값이 올 때, OnRep_Owner() 함수에 의해 오너 값이 복제가 되는 것이다. 

즉, 맨 처음에는 서버에서 빙의한 후 그 정보가 처음 복제된 NetInit이 끝나는 시점인 PostNetInit 이후에 OnRep_Owner가 실행되어 Owner가 설정된다.

### NetRole

LocalRole과 RemoteRole이 있다.

예를 들어 PlayerController의 경우,서버 기준으로 LocalRole은 Authority이고, RemoteRole은 SimulatedProxy이다.

Pawn의 경우 빙의 전에는 SimulatedProxy지만, 빙의 후에는 AutonomousProxy가 된다.

Proxy는 서버 액터를 복제한 허상을 뜻한다.

NetRole은 UENUM이다.

- ROLE_None : 게임모드는 서버에만 존재하기 때문에 프록시는 None이다.
- ROLE_Simulated**Proxy** : (클라이언트) 서버로부터 데이터를 수신하고, 이를 반영한다. 능동적으로 로직 수행하는 것은 불가능하다. 예를 들어, 다른 클라이언트의 캐릭터
- ROLE_Autonomous**Proxy** : (클라이언트) 입력 관련 로직을 능동적으로 수행할 수 있다. 입력 정보를 서버에 보낸다.
- ROLE_Authority : (서버) 게임 로직 수행 가능
- ROLE_MAX

AActor::HasAuthority : Authority를 가졌는지 확인 가능

AController::IsLocalController, APawn::IsLocallyControlled : 입력 관련 로직 수행 가능한지 확인 가능 (Authority이거나 Autonomous Proxy)

### NetDriver

용도에 따라 패킷을 처리하는 다양한 NetDriver 클래스를 제공함.

그 중 하나는 GameNetDriver. 게임 데이터를 처리하는데 사용하는 네트워크 드라이버이다.

### Channel

번치(패킷) 처리하는 여러 채널이 있다.

ControlChannel : 커넥션에 다룰 때 사용

ActorChannel : 액터 리플리케이션에 사용

VoiceChannel : 음성 데이터 전달에 사용

---

## 액터 리플리케이션

### 프로퍼티 리플리케이션

서버에서 속성 값이 **변경됐을 때** 클라이언트에 복제해 주는 것이다.

1. 액터의 bReplicates 속성을 true로 설정
2. 복제할 속성 UPROPERTY에 Replicated 키워드 설정
3. GetLifetimeReplicatedProps 함수 오버라이드해서 #include “NetUnrealNetwork.h” 헤더 파일 속 DOREPLIFETIME 매크로 사용해서 네트워크로 복제할 속성을 추가 (여기서 LIFETIME은 액터 채널의 LIFETIME을 의미함)

```cpp
//.h
UPROPERTY(Replicated)
float ServerRotationYaw;

//.cpp
void AABFountain::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME(AABFountain, ServerRotationYaw);
}  
```

위의 방식은 속성값이 언제 변경됐는지(복제됐는지) 따로 모르기 때문에 Tick에서 계속 업데이트해 줘야 함.

그래서 콜백 함수를 사용.

### OnReq_

프로퍼티 값이 변경(복제)될 때 콜백 함수를 호출하는 방법

**OnReq_ 콜백 함수는 클라이언트에서만 호출된다**. 따라서 리슨 서버의 경우, 프로퍼티 값을 변경하는 코드 부분에서 따로 OnReq_ 함수를 호출해 주면 된다. (블루프린트의 경우 서버도 호출된다)

1. UPROPERTY의 Replicated 키워드를 ReplicatedUsing 키워드로 변경
2. ReplicatedUsing에 호출할 콜백 함수를 지정
3. 호출될 콜백 함수는 UFUNCTION으로 선언해야 함

```cpp
//.h
UPROPERTY(ReplicatedUsing = OnRep_ServerRotationYaw)
float ServerRotationYaw;

UFUNCTION()
void OnRep_ServerRotationYaw();

//.cpp
void AABFountain::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME(AABFountain, ServerRotationYaw);
}  
```

### 빈도 (Frequency)

NetUpdateFrequency : 1초당 몇 번 리플리케이션을 할지. 기본값은 100.

이를 통해 리플리케이션 빈도의 최대치를 설정할 수 있다. 이는 최대치일뿐 보장되지는 않는다. 서버의 성능에 따라 서버의 Tick Rate가 달라지고, 이에 따라 리플리케이션 빈도가 달라질 수 있다. 

적응형 네트워크 업데이트(Adaptive Network Update) : 언리얼이 제공하는 기능이다. 언리얼엔진이 값이 변화하는 주기를 스스로 판단해서 빈도값을 조절해 주는 것이다. 이를 사용하기 위해서는 DefaultEngine.ini에서 [SystemSettings] net.UseAdaptiveNetUpdateFrequency = 1을 하고, 리플리케이션 빈도의 최소치를 설정해 줘야 한다(MinNetUpdateFrequency).

### 연관성(Relevancy)

서버는 서버의 관점에서 어떤 액터가 클라이언트에 연관성이 있는지 확인하는 작업을 한다. 각 클라이언트에게 레벨에 있는 일부 액터만 보이고, 들리고, 영향을 주기 때문에 서버가 클라이언트와 연관있는 액터만 모아 통신 데이터를 최소화한다.

- 리얼 뷰어(RealViewer) : 대부분 플레이어 컨트롤러를 뷰어로 한다.
- 뷰 타켓(View Target) : 플레이어 컨트롤러가 빙의한 폰
- 가해자(Instigator) : 나에게 데미지를 가한 액터
- 오너(Owner) : 최상단의 액터를 뜻한다. 액터의 오너 액터를 기준으로 연관성을 검사할 수 있다.

서버는 틱마다 각 클라이언트의 뷰어를 중심으로 모든 커넥션과 액터에 대해 연관성을 점검한다.

서버의 부하가 많이 걸린다.

연관성 판정에 대한 속성을 지정해 줄 수 있다.

- AlwaysRelevant : 항상 연관성을 가짐
- NetUseOwnerRelevancy : 자신의 연관성은 오너의 연관성으로 판정함
- OnlyRelevantToOwner : 오직 해당 Actor의 소유자(Owner)에게만 복제된다. (한 클라이언트만 볼 수 있는 것이다)
- NetCullDistanceSquared : 뷰어와의 거리에 따라 연관성 여부를 결정함 (기본값 2억2500만, 이는 제곱된 깂으로. 15000센치, 150미터이다.)

가상함수 IsNetRelevantFor()를 오버라이드해서 설정할 수 있다.

```cpp
bool AABFountain::IsNetRelevantFor(const AActor* RealViewer, const AActor* ViewTarget, const FVector& SrcLocation) const
{
	bool NetRelevantResult = Super::IsNetRelevantFor(RealViewer, ViewTarget, SrcLocation);
	if (!NetRelevantResult)
	{
		AB_LOG(LogABNetwork, Log, TEXT("Not Relevant:[%s] %s"), *RealViewer->GetName(), *SrcLocation.ToCompactString());
	}

	return NetRelevantResult;
}
```

### 우선권(NetPriority)

대역폭은 한정되어 있음. 그래서 우선권을 통해 중요한 것을 먼저 보낼 수 있음.

기본값으로 Actor는 1.0, Pawn은 3.0, PlayerController는 3.0

최종 우선권은 GetNetPriority() 가상 함수를 사용해서 계산한다. 이 함수는 업데이트를 받지 못하는 경우를 피하기 위해 NetPriority에 지난번 리플리케이션 이후 경과시간을 곱한다. 또한, 액터와 관찰자 사이의 거리와 상대적 위치도 고려해서 가중치를 곱하여 값을 조절한다.

### 휴면 상태(NetDormancy)

액터의 전송을 최소화 하기 위한 속성으로, 액터가 휴면 상태라면 연관성이 있더라도 액터 리플리케이션(RPC)을 수행하지 않는다.

- DORM_Never : 휴면 상태로 들어가지 않으며 항상 복제된다
- DORM_Awake : 휴면 상태가 아닌 상태.
- DORM_DormantAll : 모든 클라이언트에 대해 휴면 상태가 된다.
- DORM_DormantPartial : 일부 클라이언트에 대해서만 휴면 상태가 된다.
- DORM_Initial : 휴면 상태로 시작한다.

(프로퍼티 리플리케이션에는 DORM_Initial만 고려하는 것이 좋다고 함)

- `FlushNetDormancy()`: Dormant 상태를 초기화하고, 즉시 해당 액터를 복제합니다.
- `SetNetDormancy(ENetDormancy NewDormancy)`: 액터의 Dormancy 상태를 설정합니다.
- `ForceNetUpdate()`: 액터의 복제를 강제로 트리거합니다.

### 조건식 프로퍼티 리플리케이션

조건에 따라 프로퍼티를 리플리케이션에 등록하는 방법이다.

`DOREPLIFETIME_CONDITION` 매크로를 사용하면 된다.

예를 들어, 아래와 같이 COND_SimulatedOnly를 사용하면 SimulatedProxy인 클라이언트에게만 프로퍼티 리플리케이션 등록을 하게 할 수 있다. UENUM인 ELifetimeCondition에 다양한 조건들이 있다.

```cpp
void AActor::GetLifetimeReplicatedProps( TArray< FLifetimeProperty > & OutLifetimeProps ) const
{
	DOREPLIFETIME_CONDITION( AActor, ReplicatedMovement, COND_SimulatedOnly );
}
```

### 액터 리플리케이션 과정

UNetDriver::ServerReplicateActors 안에서 일어난다. 서버는 매 틱 마다 이를 실행한다.

연관성 있는 액터를 결정하고, 변경된 내용이 있는 것들을 복제한다.

현재 리플리케이션중인(`AActor::SetReplicates(true)`) 액터 각각에 대해 루프를 돌린다.

1. 휴면 상태인지 검사
2. NetUpdateFrequency 값을 검사
3. AActor::bOnlyRelevantToOwner = true면, 해당 액터가 소유된 connection의 viewer에서 AActor::IsRelevancyOwnerFor을 호출해서 해당 connection에 대해 액터의 연관성 검사를 합니다. (얘는 미리 따로 처리하는 것이다.)
4. 위 검사를 통과한 액터에 대해서 `AActor::PreReplication`을 호출한다.
5. 통과한 액터들을 ConsideredList에 추가한다.

각 connection에 대해 considered 목록 액터들의 연관성 검사를 진행한다.

1. 휴면 상태인지 검사
2. 아직 채널이 없다면 
    1. 액터가 들어있는 레벨을 클라이언트가 로드했는지 검사
    2. connection에 대해 AActor::IsNetRelevantFor로 연관성 검사

최종적으로, connection에 소유된 연관성 액터 목록이 만들어진다.

1. 이 액터들을 우선권 순으로 정렬한다.
2. 연관성 검사를 1초마다 진행해서 5초 동안 연관성이 없으면 채널을 닫는다.
3. 연관성이 있으면 채널을 연다.
4. 접속이 포화되면, 남은 액터들에 대해 포화 처리를 한다.
5. 위 모든 것을 통과한 액터에 대해 UChannel::ReplicateActor를 호출해서 액터를 Connection에 리플리케이트한다.


---

## RPC (Remote Procedure Call)

### RPC란?

- 원격 컴퓨터 프로그램에 있는 함수를 호출하는 데에 사용하는 프로토콜
- 사운드 재생, 파티클 스폰 등 액터의 핵심적인 기능과는 무관한 일시적 효과와 같은 작업을 하는 이벤트 사용을 위한 것임
- 오너십 작동 방식을 이해하는 것이 중요. 이것이 RPC 실행 장소를 결정함.

### 키워드

함수를 RPC로 선언하려면 UFUNCTION 선언에 Client, Server, NetMulticast 키워드를 붙여주면 된다.

(UE5부터는 Reliable, UnReliable도 명시해 줘야 한다. Reliable은 오류 검출 및 복구해서 느림)

- Client : 서버에서 호출되지만 클라이언트에서 실행됨
- Server : 클라이언트에서 호출되지만 서버에서 실행됨
- NetMulticast : 서버에서 호출된 다음 연결된 모든 클라이언트에서 실행됨. (클라이언트에서 호출할 수는 있지만, 이 경우 로컬에서만 실행된다!)

함수 접두사로 어떤 키워드인지 알려주는 것이 좋음

### RPC 호출 조건

- Actor에서 호출되어야 한다.
- Actor는 반드시 replicated여야 한다.
- Client RPC는 해당 Actor를 가지고 있는 Client에서만 함수가 실행된다.
- Server RPC는 클라이언트는 RPC가 호출되는 Actor를 소유해야 이를 호출할 수 있다.

### 인증 (Validation) 함수

RPC에 대한 인증 함수가 악성 파라미터를 감지한 경우, 해당 RPC를 호출한 클라이언트/서버 연결을 끊도록 시스템에 알린다.

인증 함수를 선언하려면, UFUNCTION에 WithValidation 키워드를 추가하고, Implemention 함수 옆에  Validate 함수를 만들어주면 된다.

```cpp
bool SomeRPCFunction_Validate(int32 AddHealth)
{
    if (AddHealth > MAX_ADD_HEALTH)
    {
        return false;
    }
    return true;
}

void SomeRPCFunction_Implementation(int32 AddHealth)
{
    Health += AddHealth;
}
```

### Client RPC

- 서버가 특정 클라이언트에 명령을 보낼 수 있음
- 클라이언트의 커넥션을 소유한 액터의 RPC를 사용해야 한다. 커넥션을 소유했는지는 AActor::GetNetConnection()을 통해 알 수 있다.  (액터 오너의 커넥션 반환하는 함수)


### Server RPC

- 클라이언트에서 서버로 호출하는 RPC
- 유일하게 클라이언트가 서버의 함수를 호출 할 수 있는 기능
- 서버와의 커넥션을 소유한 액터를 사용해야 함


### NetMulticast RPC

- 프로퍼티 리플리케이션과 유사하게 **연관성 기반**으로 동작함 (커넥션을 **소유하지 않아도 동작**)
- 용도는 프로퍼티 리플리케이션과 다름.
- 클라이언트에서 호출하면 호출한 클라이언트에서만 실행된다. (서버에서 호출해야 함)

### 오너쉽(소유권) 제공

서버에서 클라이언트로 오너쉽을 주는 방법

```cpp
for (FConstPlayerControllerIterator Iterator = GetWorld()->GetPlayerControllerIterator(); Iterator; ++Iterator)
{
    APlayerController* PlayerController = Iterator->Get();
    if (PlayerController && !PlayerController->IsLocalPlayerController())
		{
		    SetOwner(PlayerController);
		    break;
		}
}
```

월드에서 원하는 클래스의 액터들을 모두 가져오는 EngineUtils.h에 있는 TActorRange<>() 사용 가능

```cpp
for (APlayerController* PlayerController : TActorRange<APlayerController>(GetWorld()))
{
    if (PlayerController && !PlayerController->IsLocalPlayerController())
		{
		    SetOwner(PlayerController);
				break;
		}
}
```

오너쉽 정보는 서버가 관리한다. 서버가 아닌 클라이언트가 설정한 오너쉽은 무시된다. 클라이언트에서 변경한 소유권 정보는 서버에 복제되지 않기 때문이다.

### Reliable, Unreliable

아무래도 Reliable 과도하게 사용하면 queue 특성상 패킷 손실이 일어날 수 있다. 따라서 프레임 단위로 호출되는 함수는 Unreliable, 그리고 “총 쏘기”는 Reliable 할 필요가 있지만, 플레이어 입력에 바인딩되는 경우 입력 빈도에 제한을 둬서 조절해야 한다.

### Actor Replicate 설정

- 액터의 리플리케이트 세팅을 true로
- 해당 액터가 움직여야 하는 경우, Replicates Movement를 true로
- 해당 액터를 스폰 또는 소멸할 때는 반드시 서버에서 하기

### 그 외

- 리슨 서버의 경우는 서버가 플레이어로 참여하기 때문에 서버에서도 ServerRPC도 사용 가능.

### Property Replication vs NetMulticastRPC

차이점 : 프로퍼티 리플리케이션은 변경된 정보가 틱마다 동기화되지만, NetMulticastRPC는 이를 호출한 순간에 동기화가 되는 거임.

그래서 프로퍼티 리플리케이션은 게임에 영향을 미치는 데이터에 사용하고, NetMulticastRPC는 게임과 무관한 휘발성 데이터(효과)에 사용한다.

### 액터 컴포넌트 리플리케이션

컴포넌트의 생성자에 `SetIsReplicated(true)`으로 리플리케이션을 지정하면, 컴포넌트 준비 단계인 InitializeComponent가 끝나고 리플리케이션을 준비합니다. 준비가 완료되면 `ReadyForReplication` 를 호출합니다. 그 후 `BeginPlay()`가 진행됩니다.

정리 : InitializeComponent() → ReadyForReplication() → (액터와 액터 컴포넌트 모두)BeginPlay()

### 최적화

FVector_NetQuantize : 범위를 줄여서 데이터 전송량을 줄인다.

FVector_NetQuantizeNormal : 단위벡터에 사용하면 더 줄일 수 있다.

SimulatedProxy의 경우 서버에서 MulticastRPC를 사용하는 것보다 모든 SimulatedProxy에 대해 ClientRPC를 하는 방법으로 최적화가 가능하다. 왜? 예를 들어, 리슨서버 1, 클라 2일 때 클라가 공격한 애니메이션 실행을 MulticastRPC하게 되면 2개의 클라에 패킷을 전송해야 한다. 근데 공격한 클라에서 직접 애니메이션 재생을 하고(공격한 클라는 애니메이션 빠르게 볼 수 있음), 서버에서 공격을 한 클라를 제외한 클라에 ClientRPC하게 하면 1개의 클라에 패킷을 전송하면 된다.

(그러나 이 방법은 연관성을 고려하지 않게 된다)

## 움직임 리플리케이션

### 플레이어 움직임 리플리케이션 ( ReplicateMoveToServer() )

무브먼트 컴포넌트에 구현된 플레이어 움직임 리플리케이션 과정 (이를 반복 수행)

**AutonomousProxy**

(클라이언트)

1. 매 틱마다(UCharacterMovementComponent::TickComponent) 플레이어 입력을 감지해서 처리 ControlledCharacterMove()
2. 입력값을 토대로 움직임을 적용하고, 클라이언트의 로컬 움직임을 기록
    1. NewMove(FSavedMove_Character)에 움직이기 전 초기 상태 기록 등등(이것저것)
    2. 이동을 수행하는 PerformMovement() 실행
    3. NewMove(FSavedMove_Character)에 이동 후 상태를 기록 (FNetworkPredictionData_Client_Character 클래스 안에 있는 SavedMoves라는 TArray에 Push한다.)
3. 서버 RPC(SeverMove)를 호출해서 NewMove에 있는 일부 정보를 서버로 전송

(서버)

1. 클라이언트에게 정보를 받음
2. 입력값을 사용해서 캐릭터를 이동시키는 MoveAutonomous() 실행
3. ServerMoveHandleClientError()를 호출해서 서버에서의 결과를 클라이언트에서의 결과와 비교
4. 만약 오차가 크면 ClientAdjustPosition RPC 호출해서 클라이언트에게 수정 명령을 보냄

(클라이언트)

1. 서버가 검증한 움직임 이전 기록인, 더 이상 필요없는 움직임 기록은 SavedMoves에서 제거.
2. 명령대로 캐릭터의 위치 조정 ( ClientAdjustPosition_Implementation() )
3. bUpdatePosition = true로 만든다.  
4. 이후 ControlledCharacterMove() 함수 호출 전에 bUpdatePosition가 true면 false로 바꾸고, 서버RPC 보낸 그 이후의 움직임들에 대한 기록(SavedMoves)을 이용해서 MoveAutonomous()로 새로운 최종 위치로 움직임.

**SimulatedProxy**

서버에게 받은 움직임 정보를 부드럽게 시각적으로 표현

SimulateMovement() : 이런저런 처리들을 하고, SimulatedProxy 캐릭터의 위치, 회전, 속도 값을 저장한다. SimulatedTick()과 OnRep_ReplicateMovement에 의해 호출된다.

SmoothClientPosition() : SimulateMovement()에서 캐릭터 캡슐의 속성값을 결정하고, SmoothClientPosition()에서는 프레임 레이트에 맞게 메시를 보간해서 움직임.

### 플레이어 무브먼트 리플리케이션 디버깅

디버깅 하려면 DefaultEngine.ini에서 LogNetPlayerMovement = VeryVerbose로 설정하면 로그로 디버깅 가능, 시각화 하고 싶으면 언리얼 콘솔 명령으로 p.NetShowCorrections 1

서버에서의 오차 발생시 드로우 디버그

- 전달받은 클라이언트 위치를 붉은색으로
- 서버에서 움직인 위치를 녹색으로

오차를 전달받은 클라이언트에서의 드로우 디버그

- 클라이언트가 지정했던 위치를 붉은색으로
- 서버가 수정해준 위치를 녹색으로
- 수정은 발생했지만 서버와 클라이언트 위치가 거의 동일한 경우에는 노란색으로

### 액터 움직임 리플리케이션

RPC를 사용하지 않고, 프로퍼티 리플리케이션을 사용한다. 

ReplicatedMovement 변수

프로퍼티 리플리케이션에 ReplicatedMovement (FRepMovement) 변수의 값을 사용하는데, 이는 물리값과 움직임 정보를 기록한 멤버 변수이다. 변경되면 OnRep_ReplicatedMovement()이 호출된다. 이는 움직임과 물리값을 조정한다.

FRepMovement에는 물리값도 다루는지 여부를 알려주는 bool 값 bRepPhysics를 가진다. 이 값은  GatherCurrentMovement()에서 IsSimulatingPhysics()이고, IsWelded() == false이면 true가 된다.

물리 기록에는 FRigidBodyState 구조체를 사용한다. (위치, 회전, 속도, 각속도, 플래그)

FRepMovement::FillFrom()를 통해 현재 FRigidBodyState의 정보를 ReplicatedMovement에 저장하고,  FRepMovement::CopyTo()를 통해 복제 받은 ReplicatedMovement 정보를 FRigidBodyState로 옮긴다.

GatherCurrentMovement()

- 현재 액터의 움직임을 ReplicatedMovement 속성으로 변환해 설정하는 함수다.
- 액터의 PreReplication에서 호출된다.
- 액터의 일반 움직임과 물리 움직임을 구분해 각각 처리한다.
- 일반 움직임은 액터 위치, 회전, 속도 값이다.
- RootComponent의 IsSimulatingPhysics()일 경우, 물리 움직임도 처리한다.
- 물리를 위한 물리 씬이 따로 있다고 한다.  물리 움직임은 물리 씬에서 루트 컴포넌트의 현재 물리 상태 정보를 FRepMovement::FillFrom()을 통해 ReplicatedMovement에 옮긴다.
- 이렇게 ReplicatedMovement가 변경되면 OnRep_ReplicatedMovement()가 호출되는 것이다.
- 액터의 움직임 리플리케이션 옵션(bReplicateMovement)을 활성화해줘야 동작한다.

OnRep_ReplicatedMovement()

- 액터의 일반 움직임과 물리 움직임을 구분해 각각 처리한다.
- 일반적인 움직임에 대한 처리 : SimulatedProxy에 대해서만 처리하며, 컴포넌트의 위치와 회전 정보를 갱신한다. 속도 처리는 별도로 진행하지 않는다.
- 물리 움직임에 대한 처리 : FRepMovement::CopyTo()를 통해 ReplicatedMovement의 정보를 현재 컴포넌트의 물리 상태로 옮긴다.
- SetRigidBodyReplicatedTarget()으로 물리 씬에서 현재 컴포넌트와 일치하는 타켓(FReplicatedPhysicsTarget)을 찾아서 업데이트 한다.

ApplyRigidBodyState()

- FPhysicsReplication의 Tick()에서 호출되는 동기화 함수, 즉, 리플리케이션 될 때마다 하는 게 아니라 Tick() 마다!!
- Extrapolation해서 예측된 값을 사용해서 interpolation.
- 만약 실제 서버에서 온 값과 차이가 있으면 통신에 걸린 시간인 DeltaTime을 사용해서 에러 시간을 누적
- 에러 시간이 설정값을 넘으면 강제 조정 (HardSnap)

Static Mesh Replicate Movement 옵션 : 스태틱 메시 무브먼트 리플리케이션

## 텔레포트 구현(네트워크 게임)

### 방법론

1. 클라이언트에서만 실행 : 네트워크 게임에 부적합
2. 서버가 처리하고, 클라이언트에게 결과를 알려줌 : 지연시간 문제, 텔레포트가 아닌 Smooth한 이동이 될 수 있음.
3. 클라이언트에서 실행하고, 서버RPC도 실행해서 보정 : 클라이언트와 서버 사이에 다르게 동작할 수 있다. 왜? 만약에 텔레포트 이전 시점에서 보정이 일어난 경우, 서버는 텔레포트가 진행된 상태에서 이동 대기 목록(SavedMoves)의 것들을 실행을 한 결과물일 것이고, 클라이언트(Autonomous)는 텔레포트가 SavedMoves에 기록되는 것이 아니기 때문에, 보정이 일어난 시점의 위치로 돌아간 후 텔레포트는 하지 않고 이동 대기 목록의 것들을 실행한 결과물이 보여질 것이다.
4. CharacterMovementComponent 확장 : 텔레포트를 SavedMoves에 기록하기 때문에 보정이 일어나더라도 텔레포트가 없어지지 않는 보정이 일어난다.

### CharacterMovementComponent 확장

- 모든 언리얼 오브젝트는 초기화 오브젝트 인자(FObjectInitializer)가 있는 생성자를 사용할 수 있다.
- 초기화 오브젝트 인자를 사용해 서브 오브젝트 클래스를 변경할 수 있다.
- 이를 사용해 새로운 캐릭터 무브먼트 클래스를 생성하지 않고, 교체가 가능하다.
    - 언리얼 엔진이 지정한 캐릭터 무브먼트 컴포넌트의 이름을 사용해 해당 클래스를 찾을 수 있다.

```cpp
AABCharacterPlayer::AABCharacterPlayer(const FObjectInitializer& ObjectInitializer) : Super(ObjectInitializer.SetDefaultSubobjectClass<UABCharacterMovementComponent>(ACharacter::CharacterMovementComponentName))
```

CharacterMovementComponent 클래스에는 CompressedFlags라는 특별한 움직임에 대한 플래그가 있다. 8개 중 4개는 기능 확장을 위해 사용할 수 있다.

<div>
    <img src="/assets/images/posts_img/compressed_flags.png" alt="compressed__flags" width="100%" min-width="100px" itemprop="image">
</div>

이 플래그를 통해 서버는 클라이언트의 상황을 수시로 전달받을 수 있다.

---

### SavedMove

클라이언트 서버 사이에서 캐릭터 움직임을 동기화하는 과정에서 서버가 클라이언트에서의 캐릭터 위치를 검증했을 때, 틀리면 클라이언트의 위치를 보정해 준다.

이때, 클라이언트는 SavedMove에 저장된 데이터를 기반으로 보정 이후의 움직임들을 반영할 수 있다.

**만약 SavedMove가 없으면 어떻게 되겠는가. 서버한테 돌아온 보정 위치는 지금의 움직임이 아니라 몇 프레임 전 움직임이라 텔레포트하는 것처럼 보이는 현상이 일어난다.**

클라이언트가 서버한테 움직임 정보를 보낼 때 SavedMove에 있는 핵심 데이터를 RPC로 보낸다. 서버는 이를 기반으로 검증한다.

| 파라미터 | SavedMove에서 가져온 데이터 |
| --- | --- |
| TimeStamp | 입력 시점의 시간 |
| InAccel | 입력된 가속 벡터 |
| ClientLoc | 입력 시점의 위치 |
| CompressedMoveFlags | 비트 플래그(점프, 웅크리기, 커스텀 플래그 등) |

SavedMove에는 **CompressedFlags**가 있다. Jumped나 Crouch 등… 남은 비트(커스텀 플래그)를 개발자가 사용할 수 있다.

이동만 저장하는 게 아니라, 이런 것들도 저장해서 연결할 수 있다는 것이다.

---

### 확장 구현

[클라이언트용]

1. 커스텀 CharacterMovementComponent을 만든다.
2. 그 안에 텔레포트 함수를 구현한다.
3. PerformMovement() 함수 안에 있는 OnMovementUpdated() 함수를 오버라이드해서 조건(쿨타임 등)이 충족할 때 텔레포트 함수를 실행하게 구현.
4. 그러면 입력이 들어오면 PerformMovement()가 실행되고 조건 검사해서 텔레포트 수행

[서버용]

이제 SavedMove와 CompressedFlags를 구현해 줘야 함.

1. FNetworkPredictionData_Client를 상속 받아서 커스텀 클래스를 만들고, GetPredictionData_Client() 오버라이드 해서 FNetworkPredictionData_Client 커스텀 클래스를 반환하게 바꿈
2. FSavedMove_Character를 상속 받아서 커스텀 클래스를 만들고, 초기화 함수 Clear()와 초기값 저장하는 SetInitialPosition() 함수, 플래그 값 얻는 GetCompressedFlags() 함수 오버라이딩
3. SetInitialPosition() 함수에서 조건(쿨타임) 플래그 값을 FSavedMove_Character에 저장하고, GetCompressedFlags()에서 이 플래그 값들을 CompressedFlags 값에 넣어줌.
4. CompressedFlags를 압축 푸는 함수인 UpdateFromCompressedFlags()를 오버라이드 해서 플래그 값들을 얻어서 서버에서의 텔레포트 실행.

