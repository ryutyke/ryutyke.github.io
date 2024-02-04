---
title: "[UnrealEngine] C++ Framework"
excerpt: "[임시]언리얼 엔진이 제공하는 C++ 프레임워크"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Framework, cpp]

permalink: /unrealengine/cpp_framework/

toc: true
toc_sticky: true

date: 2024-02-05 00:10:00
last_modified_at: 2024-02-05
---
<br>

>코딩 테스트 공부하면서 막 기록한 내용... 조만간 정리 예정

# 언리얼엔진 C++ 프레임워크

언리얼 오브젝트는 언리얼 헤더 툴(Unreal Header Tool)이 관리.<br>
그래서 언리얼 헤더 툴이 알 수 있게 다 표시를 해줘야 함.<br>
언리얼 헤더 툴이 그 표시들 보고 언리얼 특유의 코드를 분석해서 intermediate에 정리. 그 후 컴파일.

<br>

## 매크로

### 필수 매크로
 
언리얼 오브젝트 선언 규칙<br>
- #include "클래스이름.generated.h" 를 include 맨 마지막에 해줘야 함.
- 클래스 선언 전에 UCLASS() 매크로 적어줘야 함.
- 클래스 이름 접두사. 액터는 A, 액터 아니면 U. 그 외 밑에 적어뒀음.
- 선언 내부에 GENERATED_BODY() 매크로도 써줘야 함. 

### 추가 매크로

- 다른 모듈에서 현재 모듈 내 클래스에 접근할 수 있게 하려면 "프로젝트 이름 대문자_API" 를 클래스 선언에 써줘야 함.
- 멤버 변수 위에는 UPROPERTY()
- 멤버 함수 위에는 UFUNCTION()

<br>

1. UClass() 속 매크로
- BlueprintType : 블루프린트에서 변수로 선언이 가능하게 함.
- Blueprintable : 블루프린트에서 이 C++ 클래스를 상속받아서 새롭게 클래스를 만들 수 있게 함.
AActor에는 이미 두 선언이 들어있음.

2. UFUNCTION() 속 매크로
- BlueprintCallable : 블루프린트에서 해당 함수 호출 가능. 대신 카테고리를 꼭 지정해줘야 함.
- BlueprintImplementableEvent : 블루프린트에서 이벤트로 지정 가능하게 
- BlueprintNativeEvent : C++, 블루프린트 모두에서 이벤트로 지정 가능

3. UPROPERTY() 속 매크로
- BlueprintReadWrite
- BlueprintReadOnly
이 둘은 blueprint에서 get() set() 할 수 있게 하는 거고,<br>
에디터 상에서. 디테일 탭에서 변수 기본 값이나 각 인스턴스들의 변수 값을 보거나 조절하고 싶다면<br>
- EditDefaultsOnly // 기본 값 (모든 인스턴스에게 적용되는 거)
- EditInstanceOnly // 각 인스턴스의 변수 값
- EditAnywhere // 모두
- VisibleDefaultsOnly
- VisibleInstanceOnly
- VisibleAnywhere

<br>

## 델리게이트
- 함수 포인터의 직접 접근이 아닌, 대리자를 통한 함수 호출 방식
- 호출할 함수나 이를 포함하는 객체가 없어져도, 대리자가 체크해 안전하게 처리할 수 있음
- 동일한 형을 가진 함수 여러 개를 대리자가 묶어서 관리하고, 필요할 때 동시에 모두 호출하는 것이 가능함.

1. DECLARE_DELEGATE_OneParam
2. DECLARE_DYNAMIC_DELEGATE
3. DECLARE_MULTICAST_DELEGATE

<br>

delegate 바인드하고 ExecuteIfBound() 또는 Broadcast()으로 바운드 된 함수들 호출. 

<br>

## 클래스 이름 접두사
변수 이름과 구분하기 위해 유형 이름을 대문자 한 글자로 나타내는 접두사를 붙입니다.<br>
예를 들어 FSkin 이 유형 이름이고, Skin 은 FSkin 의 인스턴스 입니다.
- 템플릿 클래스 접두사는 T 입니다.
- UObject 에서 상속하는 클래스 접두사는 U 입니다.
- AActor 에서 상속하는 클래스 접두사는 A 입니다.
- SWidget 에서 상속하는 클래스 접두사는 S 입니다.
- 추상 인터페이스인 클래스 접두사는 I 입니다.
- Enum (열거형)의 접두사는 E 입니다.
- Boolean (부울) 변수의 접두사는 b 입니다 (예: bPendingDestruction 또는 bHasFadedIn).
- 그 외 대부분 클래스의 접두사는 F 이나, 일부 서브시스템은 다른 글자를 사용하기도 합니다.
- Typedef 접두사는 적합한 유형을 붙여야 합니다: 구조체의 typedef 인 경우 F, UObject 등의 typedef 인 경우 U 입니다.
- 특정 템플릿 인스턴스의 typedef 는 더이상 템플릿이 아니라 그에 맞는 접두사를 붙여야 합니다, 
예: typedef TArray\<FMytype> FArrayOfMyTypes;

<br>

### 그 외의 것들
로그는 UE_LOG() 사용

<br>

언리얼 오브젝트를 생성할 땐 NewObject<클래스>();<br>
일반 C++ 오브젝트 생성할 땐 new 클래스();<br>

<br>

컴포넌트는 CreateDefaultSubobject<클래스이름>(이름) 으로 생성자에서 정적생성 가능.
RootComponent = ~~
컴포넌트인스턴스->SetupAttacchment(붙일컴포넌트인스턴스)

<br>

ConstructorHelpers::ClassFinder
ConstructorHelpers::ObjectFinder

<br>

---
[https://docs.unrealengine.com/4.26/ko/ProductionPipelines/DevelopmentSetup/CodingStandard/](https://docs.unrealengine.com/4.26/ko/ProductionPipelines/DevelopmentSetup/CodingStandard/)
