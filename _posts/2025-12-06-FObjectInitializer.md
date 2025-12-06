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
last_modified_at: 2025-12-06
---
<br>

### FObjectInitializer

FObjectInitializer는 UObject 생성 과정에서 필요한 메타데이터와 초기화 컨텍스트를 담은 임시 구조체로, UObject 생성 파이프라인에서 엔진이 생성자에 전달한다.

UObject 생성 시, (UObject는 정적 생성 안 됨)

1. 메모리 할당
2. CDO 값 복사 (BP Class는 BP CDO)
3. FObjectInitializer 생성
4. 자체 생성자 InternalConstructor 호출 (FObjectInitializer 인자)

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

A. (일단 내가 내린 결론은 정확하진 않다. 쓰면서 문제가 생기면 결론을 수정할 수도 있다.) 
UObject를 생성하면, 엔진 내부에서 FObjectInitializer 인스턴스를 만들고, 이를 ThreadContext 내부 스택에서 관리한다. 엔진은 FObjectInitializer가 필요하면 스택에서 꺼내쓰면 된다. 따라서 내가 생성자에서 FObjectInitializer를 사용하는 게 아니라면, 굳이 FObjectInitializer 버전 생성자를 쓸 필요가 없다.

하지만! UObject::CreateDefaultSubobject를 쓰는 것보다 FObjectInitializer 버전 생성자를 둬서 FObjectInitializer::CreateDefaultSubobject를 쓰는 게 아~~주 미세하게 성능이 좋을 거 같긴 하다.