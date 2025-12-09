---
title: "FObjectInitializer"
excerpt: "FObjectInitializer를 받는 생성자는 언제 써야 하는가?"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, FObjectInitializer]

permalink: /unrealengine/FObjectInitializer/

toc: true
toc_sticky: true

date: 2025-12-06 20:00:00
last_modified_at: 2025-12-09
---
<br>

### FObjectInitializer

FObjectInitializer는 UObject 생성 과정에서 필요한 메타데이터와 초기화 컨텍스트를 담은 임시 구조체로, UObject 생성 파이프라인에서 엔진이 생성자에 전달한다.  

쉽게 말하면, FObjectInitializer는 엔진이 생성자에 인자로 넘겨주는 임시 객체이고, 우리는 그것을 활용할 수 있다.  

<details>
<summary>UObject 생성 과정 접기/펼치기</summary>
<div markdown="1">

////////////////////////////////////////////////////////////////////////////////////

1. NewObject<T>() 호출 ->StaticConstructObject_Internal()

2. 메모리 할당 : StaticAllocateObject()

3. FObjectInitializer를 인자로 자체 생성자 InternalConstructor 호출. __DefaultConstructor가 호출된다. 이는 placement new를 사용하여 생성. `(*InClass->ClassConstructor)(FObjectInitializer(Result, Params));`

```cpp
static void __DefaultConstructor(const FObjectInitializer& X)
  { new((EInternal*)X.GetObj())TClass(X); }
```

4. CDO 값(프로퍼티 등) 복사

////////////////////////////////////////////////////////////////////////////////////

</div>
</details> 

<br>

최근에는, FObjectInitializer 없이도 Subobject 생성이 가능해지는 등 역할이 줄었다. 

인스턴스 생성 시, FObjectInitializer가 생성되고, ThreadContext 내부 스택에 푸시된다. `UObject::CreateDefaultSubobject()`는 해당 스택 꼭대기에 있는 FObjectInitializer를 사용해 `FObjectInitializer::CreateDefaultSubobject`를 호출한다.

```cpp
UObject* UObject::CreateDefaultSubobject(FName SubobjectFName, UClass* ReturnType, UClass* ClassToCreateByDefault, bool bIsRequired, bool bIsTransient)
{
	FObjectInitializer* CurrentInitializer = FUObjectThreadContext::Get().TopInitializer();
	UE_CLOG(!CurrentInitializer, LogObj, Fatal, TEXT("No object initializer found during construction."));
	UE_CLOG(CurrentInitializer->Obj != this, LogObj, Fatal, TEXT("Using incorrect object initializer."));
	return CurrentInitializer->CreateDefaultSubobject(this, SubobjectFName, ReturnType, ClassToCreateByDefault, bIsRequired, bIsTransient);
}
```

<br>

Q : FObjectInitializer 파라미터를 받는 생성자는 어떻게 호출되는가?

A : GENERATED_BODY 매크로에서 엔진 자체 생성자가 만들어지는데, 모든 자체 생성자는 FObjectInitializer를 받는다. 즉, UObject 기반 객체 생성 시 항상 FObjectInitializer가 생기고 이를 받는 생성자가 호출된다. 하지만 생성자 둘 중 하나는 FObjectInitializer를 사용하고, 하나는 사용하지 않는다.

```cpp
#define DEFINE_DEFAULT_CONSTRUCTOR_CALL(TClass) \
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass; }

#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass) \
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass(X); }
```

<br>

Q. 엔진은 위 둘 중 어떤 매크로를 사용할지 어떻게 결정하는가? 

A. UhtHeaderFileParser가 파싱을 통해 이를 확인한다.

```cpp
// UhtHeaderFileParser.cs

if (token.IsValue("FObjectInitializer") || token.IsValue("FPostConstructInitializeProperties"))
{
				oiCtor = true;
}

if (oiCtor && isRef && isConst)
{
				classObj.ClassExportFlags |= UhtClassExportFlags.HasObjectInitializerConstructor;
				classObj.MetaData.Add(UhtNames.ObjectInitializerConstructorDeclared, "");
}
```

```cpp
if (classObj.ClassExportFlags.HasAnyFlags(UhtClassExportFlags.HasObjectInitializerConstructor))
{
				return ConstructorType.ObjectInitializer;
}
else if (classObj.ClassExportFlags.HasAnyFlags(UhtClassExportFlags.HasDefaultConstructor))
{
				return ConstructorType.Default;
}
```

```cpp
switch (GetConstructorType(classObj))
{
	case ConstructorType.ObjectInitializer:
		if (classObj.ClassFlags.HasAnyFlags(EClassFlags.Abstract))
		{
			builder.Append("\tDEFINE_ABSTRACT_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(").Append(classObj.SourceName).Append(") \\\r\n");
		}
		else
		{
			builder.Append("\tDEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(").Append(classObj.SourceName).Append(") \\\r\n");
		}
		break;

	case ConstructorType.Default:
		if (classObj.ClassFlags.HasAnyFlags(EClassFlags.Abstract))
		{
			builder.Append("\tDEFINE_ABSTRACT_DEFAULT_CONSTRUCTOR_CALL(").Append(classObj.SourceName).Append(") \\\r\n");
		}
		else
		{
			builder.Append("\tDEFINE_DEFAULT_CONSTRUCTOR_CALL(").Append(classObj.SourceName).Append(") \\\r\n");
		}
		break;
```

<br>

Q. 그래서 언제 FObjectInitializer 버전 생성자를 써야 하는가?

A. (내가 내린 결론일뿐 정확하진 않다. 쓰면서 문제가 생기면 결론을 수정할 수도 있다.) 
UObject를 생성하면, 엔진 내부에서 FObjectInitializer 인스턴스를 만들고, 이를 ThreadContext 내부 스택에서 관리한다. 엔진은 FObjectInitializer가 필요하면 스택에서 꺼내쓰면 된다. 따라서 내가 생성자에서 FObjectInitializer를 사용하는 게 아니라면, 굳이 FObjectInitializer 버전 생성자를 쓸 필요가 없다.

<br>

FObjectInitializer를 사용하는 예시는 뭐가 있을까?

- `CreateDefaultSubobject<T>(FName SubobjectName)` : DefaultSubobject를 생성한다. 위에 적었듯, `UObject::CreateDefaultSubobject()`로 대체 가능하다.
- `DoNotCreateDefaultSubobject(FName SubobjectName)` : 현재 클래스가 상속받은 부모 클래스에서 특정 SubobjectName을 가진 기본 서브 오브젝트를 생성하지 않도록 지시한다. 부모 클래스에서 기본적으로 생성되는 컴포넌트가 자식 클래스에서는 필요 없거나 다른 방식으로 처리하고 싶을 때 주로 사용한다.
- `SetDefaultSubobjectClass<T>(FName SubobjectName, const UClass* Class)` : 부모 클래스에서 정의된 특정 서브 오브젝트의 클래스를 자식 클래스에서 다른 클래스로 오버라이드할 때 사용된다.
- `CreateOptionalDefaultSubobject<T>(FName SubobjectName)` : `CreateDefaultSubobject`와 동일하게 서브 오브젝트를 생성하지만, DoNotCreate 등 위의 함수로 인해 T 형식의 서브 오브젝트가 만들어지지 않는다면 nullptr을 반환한다.
- `CreateEditorOnlyDefaultSubobject<T>(FName SubobjectName)` : 에디터에서만 사용될 서브 오브젝트를 생성한다.

<br>

SpectatorPawn 클래스 코드이다. 이는 기존 Pawn의 MovementComponent를 없애고, MovementComponent를 자신의 것으로 변경한다. 

```cpp
ASpectatorPawn::ASpectatorPawn(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer
	.SetDefaultSubobjectClass<USpectatorPawnMovement>(Super::MovementComponentName)
	.DoNotCreateDefaultSubobject(Super::MeshComponentName)
	)
```

Character 클래스 코드이다. SetDefaultSubobjectClass나 DoNotCreateDefaultSubobject와 같은 함수로 인해 템플릿 파라미터로 들어가는 서브 오브젝트가 만들어지지 않을 경우, nullptr을 반환한다. (이는 static_cast를 사용해 구현되었다.)

```cpp
ACharacter::ACharacter(const FObjectInitializer& ObjectInitializer)
: Super(ObjectInitializer)
{

  Mesh = CreateOptionalDefaultSubobject<USkeletalMeshComponent>(ACharacter::MeshComponentName);
  if (Mesh)
  {
    // ...
  }
}
```

