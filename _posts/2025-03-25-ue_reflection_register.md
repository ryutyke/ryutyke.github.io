---
title: "UE5 리플렉션"
excerpt: "UCLASS 리플렉션 등록 과정 등등"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Reflection]

permalink: /unrealengine/reflection/

toc: true
toc_sticky: true

date: 2025-03-25 20:00:00
last_modified_at: 2025-03-25
---
<br>

### 리플렉션(Reflection)이란?

프로그램이 실행 중에 자신의 구조(예를 들어 클래스, 메서드, 필드 등)를 동적으로 조사하고 수정할 수 있는 기능입니다.

<br>

### C++에 리플렉션이 있는가?

없습니다. 그나마 C++의 RTTI는 런타임에 타입 정보를 제공하지만, 클래스, 메서드, 프로퍼티의 상세 정보를 자동으로 나열하거나 수정하는 기능은 제공하지 않습니다. 즉, 전형적인 리플렉션이 제공하는 수준의 메타데이터 접근이나 조작 기능을 갖추고 있지 않습니다.

언리얼엔진은 이처럼 C++에 없는 리플렉션을, 컴파일 전에 메타데이터(.generated.h, .gen.cpp)를 만들고, 런타임에 이를 사용하여 클래스를 나타내는 `정적 객체`를 만들어 사용하는 방법으로 직접 구현했습니다.

<br>

### 언리얼 엔진에 리플렉션이 왜 필요했을까?

1. 블루프린트/C++ 상호작용
    1. 다양한 플래그 값 및 Category 등의 메타데이터
    2. UCLASS의 프로퍼티와 함수 목록 (바인딩)
2. 프로퍼티 리플리케이트
    1. 메모리 오프셋을 통해 값 변경 확인 (DOREPLIFETIME)  
    2. 직렬화, 역직렬화 : 필드 유형, 크기 확인 및 메모리 오프셋에 직접 쓰기

이러한 기능을 위해 메타데이터들이 런타임 중에 필요합니다.

<br>

### 리플렉션 방식 (C# vs 언리얼)

*(C# 리플렉션에 대한 내용은 검색한 내용입니다. 정확하지 않을 수 있습니다.)*

**C# 리플렉션**은 메타데이터를 어셈블리에 직렬화해서 임베드합니다. 런타임에 데이터 접근 시, 역직렬화하고 메모리 구조를 구축하는 동적 분석(파싱)을 합니다. 따라서 동적 처리로 인해 오버헤드가 발생합니다.

언리얼엔진은 빌드 시간보다 **게임의 성능이 더 중요하기 때문에** 다른 방식을 사용합니다. 

**언리얼 리플렉션**은 컴파일 전에 generated.h, gen.cpp 파일을 생성해서, 컴파일 시간에 **메모리 오프셋을 고정한 정적** **메타데이터를 생성**합니다. 따라서 데이터 접근 시, 정적 데이터라 파싱 없이 포인터 연산으로 즉시 접근 가능해서 **런타임 오버헤드가 적습니다**.

(대신 이러한 이유로 런타임에 C#은 동적 타입 생성이 허용되고, 언리얼엔진은 동적 타입 생성이 안 됩니다. 그래서 블루프린트 클래스를 생성 및 수정하면, 에디터가 C++ 코드를 생성하고 프로젝트를 리빌드 하는 것입니다.)

<br>

### Unreal Header Tool(UHT)

Unreal Header Tool이 컴파일 전에 .h 파일 내 UCLASS(), UPROPERTY(), UFUNCTION() 등의 매크로를 분석해서 리플렉션에 필요한 메타데이터(.generated.h, .gen.cpp 파일)를 생성합니다.

(UhtHeaderCodeGeneratorHFile.cs, UhtHeaderCodeGeneratorCppFile.cs)

<br>

# 리플렉션 등록 과정

## 1. 메타데이터 파일 생성

- UhtHeaderCodeGeneratorHFile.cs
- UhtHeaderCodeGeneratorCppFile.cs
- ObjectMacros.h

이들이 힘을 합쳐서 .generated.h, .gen.cpp 파일을 만듭니다.

파일 안에는 런타임에 정적 객체를 생성하기 위한 데이터들이 생성됩니다.

<br>

### **UFUNCTION :**

#### <u>struct Z_Construct_UFunction_클래스이름_멤버함수이름_Statics</u>

UFUNCTION 멤버 함수당 한 개씩입니다. 함수에 대한 정보를 가지고 있습니다. **모든 데이터가 static const이고 초기화**됩니다.

- `static constexpr FMetaDataPairParam` : Category 등 값들 (`#if WITH_METADATA`)
- 각 매개 변수에 대한 `FPropertyParams`
    
    `FPropertyParams` : 각 타입에 맞는 프로퍼티 정보를 저장하는 구조체입니다. (예 : float는 `FFloatPropertyParams`, Enum은 `FEnumPropertyParams`)
    
- 모든 매개 변수의 `FPropertyParams` 포인터를 저장하는 배열 `FPropertyParamsBase* const PropPointers[]`
- `FFunctionParams` : 함수 이름, `PropPointers`, 함수 플래그, `FMetaDataPairParam` 등 이 객체 안에 있는 것들을 포함한 함수에 대한 모든 정보
    
    ```cpp
    struct FFunctionParams
    {
    	UObject*                          (*OuterFunc)();
    	UFunction*                        (*SuperFunc)();
    	const char*                         NameUTF8;
    	const char*                         OwningClassName;
    	const char*                         DelegateName;
    	const FPropertyParamsBase* const*   PropertyArray;
    	uint16                              NumProperties;
    	uint16                              StructureSize;
    	EObjectFlags                        ObjectFlags;
    	EFunctionFlags                      FunctionFlags;
    	uint16                              RPCId;
    	uint16                              RPCResponseId;
    #if WITH_METADATA
    	uint16                              NumMetaData;
    	const FMetaDataPairParam*           MetaDataArray;
    #endif
    };
    ```

<br>  

#### <u>Z_Construct_UFunction_클래스이름_함수이름()</u>

UFunction 정적 객체를 생성하는 함수입니다.

<br>

### **UCLASS :**

#### <u>Z_Construct_UClass_클래스이름_Statics</u>

UCLASS당 한 개씩입니다. 클래스에 대한 정보를 가지고 있습니다. **모든 데이터가 static const이고 초기화**됩니다.

- `static constexpr FMetaDataPairParam` : Category 등 값들 (`#if WITH_METADATA`)
- 각 멤버 변수에 대한 `FPropertyParams`
- 모든 멤버 변수의 `FPropertyParams` 포인터를 저장하는 배열 `FPropertyParamsBase* const PropPointers[]`
- 모든 멤버 함수에 대한 ‘UFunction 객체의 포인터 주소’, ‘함수 이름’을 저장하는 배열 `FuncInfo`
- 의존하는 클래스 인스턴스 생성 함수 배열 `DependentSingletons[]`
- 인터페이스 `FImplementedInterfaceParams` 배열
- `FClassParams` : StaticClass함수 주소, 클래스 플래그, `FuncInfo` , `PropPointers`, `FMetaDataPairParam` 등 이 객체 안에 있는 것들을 포함한 클래스에 대한 모든 정보  

    
    ```cpp
    struct FClassParams
    {
    	UClass*                                   (*ClassNoRegisterFunc)();
    	const char*                                 ClassConfigNameUTF8;
    	const FCppClassTypeInfoStatic*              CppClassInfo;
    	UObject*                           (*const *DependencySingletonFuncArray)();
    	const FClassFunctionLinkInfo*               FunctionLinkArray;
    	const FPropertyParamsBase* const*           PropertyArray;
    	const FImplementedInterfaceParams*          ImplementedInterfaceArray;
    	uint32                                      NumDependencySingletons : 4;
    	uint32                                      NumFunctions : 11;
    	uint32                                      NumProperties : 11;
    	uint32                                      NumImplementedInterfaces : 6;
    	uint32                                      ClassFlags; // EClassFlags
    #if WITH_METADATA
    	uint16                                      NumMetaData;
    	const FMetaDataPairParam*                   MetaDataArray;
    #endif
    };
    
    ```
<br>

#### <u>Z_Construct_UClass_클래스이름()</u>

```cpp
UClass* Z_Construct_UClass_클래스이름()
{
	if (!Z_Registration_Info_UClass_클래스이름.OuterSingleton)
	{
		UECodeGen_Private::ConstructUClass(Z_Registration_Info_UClass_클래스이름.OuterSingleton, Z_Construct_UClass_클래스이름_Statics::ClassParams);
	}
	return Z_Registration_Info_UClass_클래스이름.OuterSingleton;
}
```

UClass 정적 객체를 생성하는 함수입니다.

<br>

#### <u>DECLARE_CLASS</u>

그 외에도 `DECLARE_CLASS` 매크로를 사용해서 다양한 UCLASS 메타데이터를 생성합니다.

<details>
<summary>ObjectMacros.h의 `DECLARE_CLASS` 매크로 코드</summary>
<div markdown="1">
    
```cpp
#define DECLARE_CLASS( TClass, TSuperClass, TStaticFlags, TStaticCastFlags, TPackage, TRequiredAPI  ) \
private: \
    TClass& operator=(TClass&&);   \
    TClass& operator=(const TClass&);   \
    TRequiredAPI static UClass* GetPrivateStaticClass(); \
public: \
    /** Bitwise union of #EClassFlags pertaining to this class.*/ \
    static constexpr EClassFlags StaticClassFlags=EClassFlags(TStaticFlags); \
    /** Typedef for the base class ({{ typedef-type }}) */ \
    typedef TSuperClass Super;\
    /** Typedef for {{ typedef-type }}. */ \
    typedef TClass ThisClass;\
    /** Returns a UClass object representing this class at runtime */ \
    inline static UClass* StaticClass() \
    { \
    	return GetPrivateStaticClass(); \
    } \
    /** Returns the package this class belongs in */ \
    inline static const TCHAR* StaticPackage() \
    { \
    	return TPackage; \
    } \
    /** Returns the static cast flags for this class */ \
    inline static EClassCastFlags StaticClassCastFlags() \
    { \
    	return TStaticCastFlags; \
    } \
    /** For internal use only; use StaticConstructObject() to create new objects. */ \
    inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter = (UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags) \
    { \
    	return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags); \
    } \
    /** For internal use only; use StaticConstructObject() to create new objects. */ \
    inline void* operator new( const size_t InSize, EInternal* InMem ) \
    { \
    	return (void*)InMem; \
    } \
    /* Eliminate V1062 warning from PVS-Studio while keeping MSVC and Clang happy. */ \
    inline void operator delete(void* InMem) \
    { \
    	::operator delete(InMem); \
    }
```
</div>
</details> 

- **Super** : 부모 클래스를 정의합니다. GetSuperClass()를 통해 계층 구조를 탐색할 수 있습니다.

```cpp
typedef TSuperClass Super;\
```

- **StaticClassFlags (EClassFlags)** : 클래스의 일반적인 특성을 정의한 비트마스크 플래그. 값이 상속됩니다. (예 : IInterface면 CLASS_Interface)
<details>
<summary>코드 접기/펼치기</summary>
<div markdown="1">
        
    ```cpp
    /**
        * Flags describing a class.
        *
        * This MUST be kept in sync with EClassFlags defined in
        * Engine\Source\Programs\Shared\EpicGames.Core\UnrealEngineTypes.cs
        */
    enum EClassFlags
    {
        /** No Flags */
        CLASS_None				  = 0x00000000u,
        /** Class is abstract and can't be instantiated directly. */
        CLASS_Abstract            = 0x00000001u,
        /** Save object configuration only to Default INIs, never to local INIs. Must be combined with CLASS_Config */
        CLASS_DefaultConfig		  = 0x00000002u,
        /** Load object configuration at construction time. */
        CLASS_Config			  = 0x00000004u,
        /** This object type can't be saved; null it out at save time. */
        CLASS_Transient			  = 0x00000008u,
        /** This object type may not be available in certain context. (i.e. game runtime or in certain configuration). Optional class data is saved separately to other object types. (i.e. might use sidecar files) */
        CLASS_Optional            = 0x00000010u,
        /** */
        CLASS_MatchedSerializers  = 0x00000020u,
        /** Indicates that the config settings for this class will be saved to Project/User*.ini (similar to CLASS_GlobalUserConfig) */
        CLASS_ProjectUserConfig	  = 0x00000040u,
        /** Class is a native class - native interfaces will have CLASS_Native set, but not RF_MarkAsNative */
        CLASS_Native			  = 0x00000080u,
        /** Don't export to C++ header. */
        CLASS_NoExport UE_DEPRECATED(5.1, "CLASS_NoExport should no longer be used. It is no longer being set by engine code.") = 0x00000100u,
        /** Do not allow users to create in the editor. */
        CLASS_NotPlaceable        = 0x00000200u,
        /** Handle object configuration on a per-object basis, rather than per-class. */
        CLASS_PerObjectConfig     = 0x00000400u,
        	
        /** Whether SetUpRuntimeReplicationData still needs to be called for this class */
        CLASS_ReplicationDataIsSetUp = 0x00000800u,
        	
        /** Class can be constructed from editinline New button. */
        CLASS_EditInlineNew		  = 0x00001000u,
        /** Display properties in the editor without using categories. */
        CLASS_CollapseCategories  = 0x00002000u,
        /** Class is an interface **/
        CLASS_Interface           = 0x00004000u,
        /**  Config for this class is overridden in platform inis, reload when previewing platforms **/
        CLASS_PerPlatformConfig   = 0x00008000u,
        /** all properties and functions in this class are const and should be exported as const */
        CLASS_Const			      = 0x00010000u,
        
        /** Class flag indicating objects of this class need deferred dependency loading */
        CLASS_NeedsDeferredDependencyLoading = 0x00020000u,
        	
        /** Indicates that the class was created from blueprint source material */
        CLASS_CompiledFromBlueprint  = 0x00040000u,
        
        /** Indicates that only the bare minimum bits of this class should be DLL exported/imported */
        CLASS_MinimalAPI	      = 0x00080000u,
        	
        /** Indicates this class must be DLL exported/imported (along with all of it's members) */
        CLASS_RequiredAPI	      = 0x00100000u,
        
        /** Indicates that references to this class default to instanced. Used to be subclasses of UComponent, but now can be any UObject */
        CLASS_DefaultToInstanced  = 0x00200000u,
        
        /** Indicates that the parent token stream has been merged with ours. */
        CLASS_TokenStreamAssembled  = 0x00400000u,
        /** Class has component properties. */
        CLASS_HasInstancedReference= 0x00800000u,
        /** Don't show this class in the editor class browser or edit inline new menus. */
        CLASS_Hidden			  = 0x01000000u,
        /** Don't save objects of this class when serializing */
        CLASS_Deprecated		  = 0x02000000u,
        /** Class not shown in editor drop down for class selection */
        CLASS_HideDropDown		  = 0x04000000u,
        /** Class settings are saved to <AppData>/..../Blah.ini (as opposed to CLASS_DefaultConfig) */
        CLASS_GlobalUserConfig	  = 0x08000000u,
        /** Class was declared directly in C++ and has no boilerplate generated by UnrealHeaderTool */
        CLASS_Intrinsic			  = 0x10000000u,
        /** Class has already been constructed (maybe in a previous DLL version before hot-reload). */
        CLASS_Constructed		  = 0x20000000u,
        /** Indicates that object configuration will not check against ini base/defaults when serialized */
        CLASS_ConfigDoNotCheckDefaults = 0x40000000u,
        /** Class has been consigned to oblivion as part of a blueprint recompile, and a newer version currently exists. */
        CLASS_NewerVersionExists  = 0x80000000u,
    };
    ```
</div>
</details> 

- **StaticClassCastFlags (EClassCastFlags)** : **핵심 클래스의 빠른 타입 캐스팅**을 위해 사용되는 비트마스크 플래그. 값이 상속됩니다. **Cast<>()**에 사용됩니다. (예 : AActor 상속 받았으면 CASTCLASS_AActor)

<details>
<summary>코드 접기/펼치기</summary>
<div markdown="1">
        
    ```cpp
    /**
        * Flags used for quickly casting classes of certain types; all class cast flags are inherited
        *
        * This MUST be kept in sync with EClassCastFlags defined in
        * Engine\Source\Programs\Shared\EpicGames.Core\UnrealEngineTypes.cs
        */
    enum EClassCastFlags : uint64
    {
        CASTCLASS_None = 0x0000000000000000,
        
        CASTCLASS_UField						= 0x0000000000000001,
        CASTCLASS_FInt8Property					= 0x0000000000000002,
        CASTCLASS_UEnum							= 0x0000000000000004,
        CASTCLASS_UStruct						= 0x0000000000000008,
        CASTCLASS_UScriptStruct					= 0x0000000000000010,
        CASTCLASS_UClass						= 0x0000000000000020,
        CASTCLASS_FByteProperty					= 0x0000000000000040,
        CASTCLASS_FIntProperty					= 0x0000000000000080,
        CASTCLASS_FFloatProperty				= 0x0000000000000100,
        CASTCLASS_FUInt64Property				= 0x0000000000000200,
        CASTCLASS_FClassProperty				= 0x0000000000000400,
        CASTCLASS_FUInt32Property				= 0x0000000000000800,
        CASTCLASS_FInterfaceProperty			= 0x0000000000001000,
        CASTCLASS_FNameProperty					= 0x0000000000002000,
        CASTCLASS_FStrProperty					= 0x0000000000004000,
        CASTCLASS_FProperty						= 0x0000000000008000,
        CASTCLASS_FObjectProperty				= 0x0000000000010000,
        CASTCLASS_FBoolProperty					= 0x0000000000020000,
        CASTCLASS_FUInt16Property				= 0x0000000000040000,
        CASTCLASS_UFunction						= 0x0000000000080000,
        CASTCLASS_FStructProperty				= 0x0000000000100000,
        CASTCLASS_FArrayProperty				= 0x0000000000200000,
        CASTCLASS_FInt64Property				= 0x0000000000400000,
        CASTCLASS_FDelegateProperty				= 0x0000000000800000,
        CASTCLASS_FNumericProperty				= 0x0000000001000000,
        CASTCLASS_FMulticastDelegateProperty	= 0x0000000002000000,
        CASTCLASS_FObjectPropertyBase			= 0x0000000004000000,
        CASTCLASS_FWeakObjectProperty			= 0x0000000008000000,
        CASTCLASS_FLazyObjectProperty			= 0x0000000010000000,
        CASTCLASS_FSoftObjectProperty			= 0x0000000020000000,
        CASTCLASS_FTextProperty					= 0x0000000040000000,
        CASTCLASS_FInt16Property				= 0x0000000080000000,
        CASTCLASS_FDoubleProperty				= 0x0000000100000000,
        CASTCLASS_FSoftClassProperty			= 0x0000000200000000,
        CASTCLASS_UPackage						= 0x0000000400000000,
        CASTCLASS_ULevel						= 0x0000000800000000,
        CASTCLASS_AActor						= 0x0000001000000000,
        CASTCLASS_APlayerController				= 0x0000002000000000,
        CASTCLASS_APawn							= 0x0000004000000000,
        CASTCLASS_USceneComponent				= 0x0000008000000000,
        CASTCLASS_UPrimitiveComponent			= 0x0000010000000000,
        CASTCLASS_USkinnedMeshComponent			= 0x0000020000000000,
        CASTCLASS_USkeletalMeshComponent		= 0x0000040000000000,
        CASTCLASS_UBlueprint					= 0x0000080000000000,
        CASTCLASS_UDelegateFunction				= 0x0000100000000000,
        CASTCLASS_UStaticMeshComponent			= 0x0000200000000000,
        CASTCLASS_FMapProperty					= 0x0000400000000000,
        CASTCLASS_FSetProperty					= 0x0000800000000000,
        CASTCLASS_FEnumProperty					= 0x0001000000000000,
        CASTCLASS_USparseDelegateFunction			= 0x0002000000000000,
        CASTCLASS_FMulticastInlineDelegateProperty	= 0x0004000000000000,
        CASTCLASS_FMulticastSparseDelegateProperty	= 0x0008000000000000,
        CASTCLASS_FFieldPathProperty			= 0x0010000000000000,
        CASTCLASS_FLargeWorldCoordinatesRealProperty = 0x0080000000000000,
        CASTCLASS_FOptionalProperty				= 0x0100000000000000,
        CASTCLASS_FVerseValueProperty			= 0x0200000000000000,
        CASTCLASS_UVerseVMClass					= 0x0400000000000000,
    };
    ```
</div>
</details> 

- **StaticPackage (TCHAR*)** : 클래스가 속한 패키지. (예 : TEXT(”/Script/Engine”) )

- **StaticClass() :** 클래스의 UClass 인스턴스를 반환하는 정적 함수입니다. UClass 인스턴스는 싱글톤으로 관리됩니다. 처음 호출될 때 싱글톤 인스턴스가 생성됩니다.

```cpp
/** Returns a UClass object representing this class at runtime */ \
inline static UClass* StaticClass() \
{ \
	return GetPrivateStaticClass(); \
} \
```

<br>

### **UPROPERTY :**

프로퍼티에 대한 다양한 정보(이름, OnRep 함수 이름, 플래그, 오프셋 등)를 각 타입에 맞는 static const 구조체(FPropertyParams)에 보관합니다. 

```cpp
struct FPropertyParamsBaseWithOffset // : FPropertyParamsBase
{
	const char*    NameUTF8;
	const char*       RepNotifyFuncUTF8;
	EPropertyFlags    PropertyFlags;
	EPropertyGenFlags Flags;
	EObjectFlags   ObjectFlags;
	SetterFuncPtr  SetterFunc;
	GetterFuncPtr  GetterFunc;
	uint16         ArrayDim;
	uint16         Offset;
};
```

Enum, Struct, Float, Int, Map 등 종류에 따라 다양한 FPropertyParams가 있습니다.

그 중에, FStructPropertyParams을 예시로 보면

```cpp
// UObjectGlobals.h
	
struct FStructPropertyParams // : FPropertyParamsBaseWithOffset
{
	const char*      NameUTF8;
	const char*         RepNotifyFuncUTF8;
	EPropertyFlags      PropertyFlags;
	EPropertyGenFlags   Flags;
	EObjectFlags     ObjectFlags;
	SetterFuncPtr  SetterFunc;
	GetterFuncPtr  GetterFunc;
	uint16           ArrayDim;
	uint16           Offset;
	UScriptStruct* (*ScriptStructFunc)();
#if WITH_METADATA
	uint16                              NumMetaData;
	const FMetaDataPairParam*           MetaDataArray;
#endif
};
```

```cpp
static const UECodeGen_Private::FStructPropertyParams NewProp_CurrentStat;
const UECodeGen_Private::FStructPropertyParams Z_Construct_UClass_UDSStatComponent_Statics::NewProp_CurrentStat = { "CurrentStat", "OnRep_CurrentStat", (EPropertyFlags)0x0020080100022035, UECodeGen_Private::EPropertyGenFlags::Struct, RF_Public|RF_Transient|RF_MarkAsNative, nullptr, nullptr, 1, STRUCT_OFFSET(UDSStatComponent, CurrentStat), Z_Construct_UScriptStruct_FDSCharacterStat, METADATA_PARAMS(UE_ARRAY_COUNT(NewProp_CurrentStat_MetaData), NewProp_CurrentStat_MetaData) }; // 519389729

```

- EPropertyGenFlags: 프로퍼티 타입 (Float, Double 등)
<details>
<summary>코드 접기/펼치기</summary>
<div markdown="1">
        
```cpp
enum class EPropertyGenFlags : uint8
{
    None              = 0x00,
        
    // First 6 bits are the property type
    Byte              = 0x00,
    Int8              = 0x01,
    Int16             = 0x02,
    Int               = 0x03,
    Int64             = 0x04,
    UInt16            = 0x05,
    UInt32            = 0x06,
    UInt64            = 0x07,
    //                = 0x08,
    //                = 0x09,
    Float             = 0x0A,
    Double            = 0x0B,
    Bool              = 0x0C,
    SoftClass         = 0x0D,
    WeakObject        = 0x0E,
    LazyObject        = 0x0F,
    SoftObject        = 0x10,
    Class             = 0x11,
    Object            = 0x12,
    Interface         = 0x13,
    Name              = 0x14,
    Str               = 0x15,
    Array             = 0x16,
    Map               = 0x17,
    Set               = 0x18,
    Struct            = 0x19,
    Delegate          = 0x1A,
    InlineMulticastDelegate = 0x1B,
    SparseMulticastDelegate = 0x1C,
    Text              = 0x1D,
    Enum              = 0x1E,
    FieldPath         = 0x1F,
    LargeWorldCoordinatesReal = 0x20,
    Optional          = 0x21,
    VValue            = 0x22,
        
    // Property-specific flags
    NativeBool        = 0x40,
    ObjectPtr         = 0x40,
        
};
```

</div>
</details> 
        
- EPropertyFlags : 프로퍼티에 대한 플래그 (Edit, BlueprintVisible, Transient 등)

<details>
<summary>코드 접기/펼치기</summary>
<div markdown="1">
        
```cpp
// ObjectMacros.h
        
/**
    * Flags associated with each property in a class, overriding the
    * property's default behavior.
    * @warning When adding one here, please update ParsePropertyFlags()
    * 
    * This MUST be kept in sync with EPackageFlags defined in
    * Engine\Source\Programs\Shared\EpicGames.Core\UnrealEngineTypes.cs
    */
enum EPropertyFlags : uint64
{
    CPF_None = 0,
        
    CPF_Edit							= 0x0000000000000001,	///< Property is user-settable in the editor.
    CPF_ConstParm						= 0x0000000000000002,	///< This is a constant function parameter
    CPF_BlueprintVisible				= 0x0000000000000004,	///< This property can be read by blueprint code
    CPF_ExportObject					= 0x0000000000000008,	///< Object can be exported with actor.
    CPF_BlueprintReadOnly				= 0x0000000000000010,	///< This property cannot be modified by blueprint code
    CPF_Net								= 0x0000000000000020,	///< Property is relevant to network replication.
    CPF_EditFixedSize					= 0x0000000000000040,	///< Indicates that elements of an array can be modified, but its size cannot be changed.
    CPF_Parm							= 0x0000000000000080,	///< Function/When call parameter.
    CPF_OutParm							= 0x0000000000000100,	///< Value is copied out after function call.
    CPF_ZeroConstructor					= 0x0000000000000200,	///< memset is fine for construction
    CPF_ReturnParm						= 0x0000000000000400,	///< Return value.
    CPF_DisableEditOnTemplate			= 0x0000000000000800,	///< Disable editing of this property on an archetype/sub-blueprint
    CPF_NonNullable						= 0x0000000000001000,	///< Object property can never be null
    CPF_Transient   					= 0x0000000000002000,	///< Property is transient: shouldn't be saved or loaded, except for Blueprint CDOs.
    CPF_Config      					= 0x0000000000004000,	///< Property should be loaded/saved as permanent profile.
    CPF_RequiredParm					= 0x0000000000008000,	///< Parameter must be linked explicitly in blueprint. Leaving the parameter out results in a compile error. 
    CPF_DisableEditOnInstance			= 0x0000000000010000,	///< Disable editing on an instance of this class
    CPF_EditConst   					= 0x0000000000020000,	///< Property is uneditable in the editor.
    CPF_GlobalConfig					= 0x0000000000040000,	///< Load config from base class, not subclass.
    CPF_InstancedReference				= 0x0000000000080000,	///< Property is a component references.
    //CPF_								= 0x0000000000100000,	///<
    CPF_DuplicateTransient				= 0x0000000000200000,	///< Property should always be reset to the default value during any type of duplication (copy/paste, binary duplication, etc.)
    //CPF_								= 0x0000000000400000,	///< 
    //CPF_    							= 0x0000000000800000,	///< 
    CPF_SaveGame						= 0x0000000001000000,	///< Property should be serialized for save games, this is only checked for game-specific archives with ArIsSaveGame
    CPF_NoClear							= 0x0000000002000000,	///< Hide clear button.
    //CPF_  							= 0x0000000004000000,	///<
    CPF_ReferenceParm					= 0x0000000008000000,	///< Value is passed by reference; CPF_OutParam and CPF_Param should also be set.
    CPF_BlueprintAssignable				= 0x0000000010000000,	///< MC Delegates only.  Property should be exposed for assigning in blueprint code
    CPF_Deprecated  					= 0x0000000020000000,	///< Property is deprecated.  Read it from an archive, but don't save it.
    CPF_IsPlainOldData					= 0x0000000040000000,	///< If this is set, then the property can be memcopied instead of CopyCompleteValue / CopySingleValue
    CPF_RepSkip							= 0x0000000080000000,	///< Not replicated. For non replicated properties in replicated structs 
    CPF_RepNotify						= 0x0000000100000000,	///< Notify actors when a property is replicated
    CPF_Interp							= 0x0000000200000000,	///< interpolatable property for use with cinematics
    CPF_NonTransactional				= 0x0000000400000000,	///< Property isn't transacted
    CPF_EditorOnly						= 0x0000000800000000,	///< Property should only be loaded in the editor
    CPF_NoDestructor					= 0x0000001000000000,	///< No destructor
    //CPF_								= 0x0000002000000000,	///<
    CPF_AutoWeak						= 0x0000004000000000,	///< Only used for weak pointers, means the export type is autoweak
    CPF_ContainsInstancedReference		= 0x0000008000000000,	///< Property contains component references.
    CPF_AssetRegistrySearchable			= 0x0000010000000000,	///< asset instances will add properties with this flag to the asset registry automatically
    CPF_SimpleDisplay					= 0x0000020000000000,	///< The property is visible by default in the editor details view
    CPF_AdvancedDisplay					= 0x0000040000000000,	///< The property is advanced and not visible by default in the editor details view
    CPF_Protected						= 0x0000080000000000,	///< property is protected from the perspective of script
    CPF_BlueprintCallable				= 0x0000100000000000,	///< MC Delegates only.  Property should be exposed for calling in blueprint code
    CPF_BlueprintAuthorityOnly			= 0x0000200000000000,	///< MC Delegates only.  This delegate accepts (only in blueprint) only events with BlueprintAuthorityOnly.
    CPF_TextExportTransient				= 0x0000400000000000,	///< Property shouldn't be exported to text format (e.g. copy/paste)
    CPF_NonPIEDuplicateTransient		= 0x0000800000000000,	///< Property should only be copied in PIE
    CPF_ExposeOnSpawn					= 0x0001000000000000,	///< Property is exposed on spawn
    CPF_PersistentInstance				= 0x0002000000000000,	///< A object referenced by the property is duplicated like a component. (Each actor should have an own instance.)
    CPF_UObjectWrapper					= 0x0004000000000000,	///< Property was parsed as a wrapper class like TSubclassOf<T>, FScriptInterface etc., rather than a USomething*
    CPF_HasGetValueTypeHash				= 0x0008000000000000,	///< This property can generate a meaningful hash value.
    CPF_NativeAccessSpecifierPublic		= 0x0010000000000000,	///< Public native access specifier
    CPF_NativeAccessSpecifierProtected	= 0x0020000000000000,	///< Protected native access specifier
    CPF_NativeAccessSpecifierPrivate	= 0x0040000000000000,	///< Private native access specifier
    CPF_SkipSerialization				= 0x0080000000000000,	///< Property shouldn't be serialized, can still be exported to text
    CPF_TObjectPtr						= 0x0100000000000000,	///< Property is a TObjectPtr<T> instead of a USomething*. Need to differentiate between TObjectclassOf and TObjectPtr
    CPF_ExperimentalOverridableLogic	= 0x0200000000000000,	///< ****Experimental*** Property will use different logic to serialize knowing what changes are done against its default use the overridable information provided by the overridable manager on the object
    CPF_ExperimentalAlwaysOverriden		= 0x0400000000000000,	///< ****Experimental*** Property should never inherit from the parent when using overridable serialization
};
```
</div>
</details> 

- EObjectFlags : 오브젝트에 대한 플래그 (Standalone, Transient 등)

<details>
<summary>코드 접기/펼치기</summary>
<div markdown="1">
        
```cpp
/**
    * Flags describing an object instance
    * When modifying this enum, update the LexToString implementation! 
    */
enum EObjectFlags
{
    // Do not add new flags unless they truly belong here. There are alternatives.
    // if you change any the bit of any of the RF_Load flags, then you will need legacy serialization
    RF_NoFlags					= 0x00000000,	///< No flags, used to avoid a cast
        
    // This first group of flags mostly has to do with what kind of object it is. Other than transient, these are the persistent object flags.
    // The garbage collector also tends to look at these.
    RF_Public					=0x00000001,	///< Object is visible outside its package.
    RF_Standalone				=0x00000002,	///< Keep object around for editing even if unreferenced.
    RF_MarkAsNative				=0x00000004,	///< Object (UField) will be marked as native on construction (DO NOT USE THIS FLAG in HasAnyFlags() etc)
    RF_Transactional			=0x00000008,	///< Object is transactional.
    RF_ClassDefaultObject		=0x00000010,	///< This object is used as the default template for all instances of a class. One object is created for each class
    RF_ArchetypeObject			=0x00000020,	///< This object can be used as a template for instancing objects. This is set on all types of object templates
    RF_Transient				=0x00000040,	///< Don't save object.
        
    // This group of flags is primarily concerned with garbage collection.
    RF_MarkAsRootSet			=0x00000080,	///< Object will be marked as root set on construction and not be garbage collected, even if unreferenced (DO NOT USE THIS FLAG in HasAnyFlags() etc)
    RF_TagGarbageTemp			=0x00000100,	///< This is a temp user flag for various utilities that need to use the garbage collector. The garbage collector itself does not interpret it.
        
    // The group of flags tracks the stages of the lifetime of a uobject
    RF_NeedInitialization		=0x00000200,	///< This object has not completed its initialization process. Cleared when ~FObjectInitializer completes
    RF_NeedLoad					=0x00000400,	///< During load, indicates object needs loading.
    RF_KeepForCooker			=0x00000800,	///< Keep this object during garbage collection because it's still being used by the cooker
    RF_NeedPostLoad				=0x00001000,	///< Object needs to be postloaded.
    RF_NeedPostLoadSubobjects	=0x00002000,	///< During load, indicates that the object still needs to instance subobjects and fixup serialized component references
    RF_NewerVersionExists		=0x00004000,	///< Object has been consigned to oblivion due to its owner package being reloaded, and a newer version currently exists
    RF_BeginDestroyed			=0x00008000,	///< BeginDestroy has been called on the object.
    RF_FinishDestroyed			=0x00010000,	///< FinishDestroy has been called on the object.
        
    // Misc. Flags
    RF_BeingRegenerated			=0x00020000,	///< Flagged on UObjects that are used to create UClasses (e.g. Blueprints) while they are regenerating their UClass on load (See FLinkerLoad::CreateExport()), as well as UClass objects in the midst of being created
    RF_DefaultSubObject			=0x00040000,	///< Flagged on subobject templates that were created in a class constructor, and all instances created from those templates
    RF_WasLoaded				=0x00080000,	///< Flagged on UObjects that were loaded
    RF_TextExportTransient		=0x00100000,	///< Do not export object to text form (e.g. copy/paste). Generally used for sub-objects that can be regenerated from data in their parent object.
    RF_LoadCompleted			=0x00200000,	///< Object has been completely serialized by linkerload at least once. DO NOT USE THIS FLAG, It should be replaced with RF_WasLoaded.
    RF_InheritableComponentTemplate = 0x00400000, ///< Flagged on subobject templates stored inside a class instead of the class default object, they are instanced after default subobjects
    RF_DuplicateTransient		=0x00800000,	///< Object should not be included in any type of duplication (copy/paste, binary duplication, etc.)
    RF_StrongRefOnFrame			=0x01000000,	///< References to this object from persistent function frame are handled as strong ones.
    RF_NonPIEDuplicateTransient	=0x02000000,	///< Object should not be included for duplication unless it's being duplicated for a PIE session
    // RF_Dynamic				=0x04000000,	///< Was removed along with bp nativization
    RF_WillBeLoaded				=0x08000000,	///< This object was constructed during load and will be loaded shortly
    RF_HasExternalPackage		=0x10000000,	///< This object has an external package assigned and should look it up when getting the outermost package
    RF_HasPlaceholderType		=0x20000000,	///< This object was instanced from a placeholder type (e.g. on load). References to it are serialized but externally resolve to NULL from a logical point of view (for type safety).
        
    // RF_MirroredGarbage is mirrored in EInternalObjectFlags::Garbage because checking the internal flags is much faster for the Garbage Collector
    // while checking the object flags is much faster outside of it where the Object pointer is already available and most likely cached.
    RF_MirroredGarbage			=0x40000000,	///< Garbage from logical point of view and should not be referenced. This flag is mirrored in EInternalObjectFlags as Garbage for performance
    RF_AllocatedInSharedPage	=0x80000000,	///< Allocated from a ref-counted page shared with other UObjects
};
```
</div>
</details> 

- 메모리 오프셋
    
예 : `STRUCT_OFFSET(UDSStatComponent, CurrentStat)`
    
```cpp
#define STRUCT_OFFSET( struc, member )	offsetof(struc, member)
#define offsetof(s,m) ((::size_t)&reinterpret_cast<char const volatile&>((((s*)0)->m)))
```

<br>    

### 공통적으로,

`#if WITH_METADATA (Editor 환경이고, Editor Only Data가 있을 때)`

FMetaDataPairParam  : Tooltip, category 등 키-값 쌍 메타데이터를 보관합니다.

```cpp
// 런타임에서 Health 프로퍼티의 메타데이터 조회
if (UProperty* Prop = FindField<UProperty>(AMyActor::StaticClass(), "Health")) {
    FString Category = Prop->GetMetaData("Category"); // "Gameplay"
    FString Tooltip = Prop->GetMetaData("Tooltip");   // "체력 값입니다."
}
```

---

## 2. Registrations에 등록

생성자에서 일어나는 과정입니다.

`Z_Construct_UClass_클래스이름` 함수는 gen.cpp에 있는 FClassRegisterCompiledInInfo ClassInfo[]에 첫 번째 인자로 들어갑니다. 또한, 이것은 FRegisterCompiledInInfo에 들어갑니다.

```cpp
// Begin Registration
struct Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics
{
	static constexpr FStructRegisterCompiledInInfo ScriptStructInfo[] = {
		{ FBuffEntry::StaticStruct, Z_Construct_UScriptStruct_FBuffEntry_Statics::NewStructOps, TEXT("BuffEntry"), &Z_Registration_Info_UScriptStruct_BuffEntry, CONSTRUCT_RELOAD_VERSION_INFO(FStructReloadVersionInfo, sizeof(FBuffEntry), 3729973798U) },
	};
	static constexpr FClassRegisterCompiledInInfo ClassInfo[] = {
		{ Z_Construct_UClass_UDSStatComponent, UDSStatComponent::StaticClass, TEXT("UDSStatComponent"), &Z_Registration_Info_UClass_UDSStatComponent, CONSTRUCT_RELOAD_VERSION_INFO(FClassReloadVersionInfo, sizeof(UDSStatComponent), 522095325U) },
	};
};
static FRegisterCompiledInInfo Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_3883540658(TEXT("/Script/Project25L"),
	Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ClassInfo, UE_ARRAY_COUNT(Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ClassInfo),
	Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ScriptStructInfo, UE_ARRAY_COUNT(Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ScriptStructInfo),
	nullptr, 0);
// End Registration
```

  
#### <u>static FRegisterCompiledInInfo</u>

해당 파일에 있는 UCLASS, USTRUCT, UENUM 등록을 위한 정보로(ClassInfo, ScriptStructInfo, 클래스 개수, 구조체 개수)가 있습니다. 

```cpp
static FRegisterCompiledInInfo Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_3883540658(TEXT("/Script/Project25L"),
	Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ClassInfo, UE_ARRAY_COUNT(Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ClassInfo),
	Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ScriptStructInfo, UE_ARRAY_COUNT(Z_CompiledInDeferFile_FID_Perforce_25L_Project25L_Source_Project25L_Stat_DSStatComponent_h_Statics::ScriptStructInfo),
	nullptr, 0);
```

FRegisterCompiledInInfo **생성자**에서  RegisterCompiledInInfo를 호출해서,

```cpp
/**
 * Helper class to perform registration of object information.  It blindly forwards a call to RegisterCompiledInInfo
 */
struct FRegisterCompiledInInfo
{
	template <typename ... Args>
	FRegisterCompiledInInfo(Args&& ... args)
	{
		RegisterCompiledInInfo(std::forward<Args>(args)...);
	}
};
```

```cpp
// Multiple registrations
void RegisterCompiledInInfo(const TCHAR* PackageName, const FClassRegisterCompiledInInfo* ClassInfo, size_t NumClassInfo, const FStructRegisterCompiledInInfo* StructInfo, size_t NumStructInfo, const FEnumRegisterCompiledInInfo* EnumInfo, size_t NumEnumInfo)
{
	LLM_SCOPE(ELLMTag::UObject);

	for (size_t Index = 0; Index < NumClassInfo; ++Index)
	{
		const FClassRegisterCompiledInInfo& Info = ClassInfo[Index];
		RegisterCompiledInInfo(Info.OuterRegister, Info.InnerRegister, PackageName, Info.Name, *Info.Info, Info.VersionInfo);
	}

	// ... struct 등록

	// ... enum 등록
}
```

```cpp
static constexpr FClassRegisterCompiledInInfo ClassInfo[] = {
	{ Z_Construct_UClass_UDSStatComponent, UDSStatComponent::StaticClass, TEXT("UDSStatComponent"), &Z_Registration_Info_UClass_UDSStatComponent, CONSTRUCT_RELOAD_VERSION_INFO(FClassReloadVersionInfo, sizeof(UDSStatComponent), 522095325U) },
};
```

FClassDeferredRegistry에 있는 TDeferredRegistry 타입 싱글톤 변수를 가져와서 AddRegistration 함수를 호출합니다. (클래스는 클래스에, 구조체는 구조체에, 열거형은 열겨형에)

```cpp
void RegisterCompiledInInfo(class UClass* (*InOuterRegister)(), class UClass* (*InInnerRegister)(), const TCHAR* InPackageName, const TCHAR* InName, FClassRegistrationInfo& InInfo, const FClassReloadVersionInfo& InVersionInfo)
{
	// ...
	FClassDeferredRegistry::AddResult result = FClassDeferredRegistry::Get().AddRegistration(InOuterRegister, InInnerRegister, InPackageName, InName, InInfo, InVersionInfo);
	// ...
}
```

AddRegistration()은 Reload 여부 및 버전 변경 여부를 확인하고 `TArray<FRegistrant> Registrations`에 추가합니다.

```cpp
AddResult AddRegistration(TType* (*InOuterRegister)(), TType* (*InInnerRegister)(), const TCHAR* InPackageName, const TCHAR* InName, TInfo& InInfo, const TVersion& InVersion)
{
#if WITH_RELOAD
	// ...
	if (bAdd)
	{
		Registrations.Add(FRegistrant{ InOuterRegister, InInnerRegister, InPackageName, &InInfo, OldSingleton, bHasChanged });
	}
	// ...
#else
	Registrations.Add(FRegistrant{ InOuterRegister, InInnerRegister, InPackageName, &InInfo });

}
```

<br>

결과 :

- “Registrations”에 리플렉션 등록에 필요한 정보들(등록 함수 포인터, 클래스 info 등)이 등록됩니다.

---

## 3. InnerRegister

`FCoreUObjectModule`의 `StartupModule()`   

와  

`FEngineLoop.PreInitPostStartupScreen()` → `ProcessNewlyLoadedUObjects()`  

에서

`void UClassRegisterAllCompiledInClasses()` 에서 `InnerRegister(Registrant)`가 호출됩니다.

```cpp
/** Register all loaded classes */
void UClassRegisterAllCompiledInClasses()
{
#if WITH_RELOAD
	TArray<UClass*> AddedClasses;
#endif
	SCOPED_BOOT_TIMING("UClassRegisterAllCompiledInClasses");
	LLM_SCOPE(ELLMTag::UObject);

	FClassDeferredRegistry& Registry = FClassDeferredRegistry::Get();

	Registry.ProcessChangedObjects();

	for (const FClassDeferredRegistry::FRegistrant& Registrant : Registry.GetRegistrations())
	{
		UClass* RegisteredClass = FClassDeferredRegistry::InnerRegister(Registrant);
	
	// ...
}
```

```cpp
struct FRegistrant
{
	TType* (*OuterRegisterFn)();
	TType* (*InnerRegisterFn)();
	const TCHAR* PackageName;
	TInfo* Info;
	// ...
}
```

InnerRegisterFn은 코드를 위로 거슬러 올라가보면 `UDSStatComponent::StaticClass`입니다.

```cpp
OuterRegister : Z_Construct_UClass_UDSStatComponent
InnerRegister : UDSStatComponent::StaticClass
PackageName : TEXT("/Script/Project25L")
Name : TEXT("UDSStatComponent")
Info : &Z_Registration_Info_UClass_UDSStatComponent
VersionInfo : CONSTRUCT_RELOAD_VERSION_INFO(FClassReloadVersionInfo, sizeof(UDSStatComponent), 522095325U)
```

StaticClass()를 호출하면, GetPrivateStaticClass()가 호출됩니다.

```cpp
/** Returns a UClass object representing this class at runtime */ \
inline static UClass* StaticClass() \
{ \
	return GetPrivateStaticClass(); \
} \
```

`GetPrivateStaticClass()`는  gen.cpp에 있는 `IMPLEMENT_CLASS_NO_AUTO_REGISTRATION` 매크로 안에 있는데, 클래스의 `Z_Registration_Info_UClass_` 싱글톤 인스턴스가 없다면  `GetPrivateStaticClassBody()` 함수를 호출해서 인스턴스를 만들어 반환하고, 이미 싱글톤으로 관리되고 있다면 그것을 반환합니다. 

<details>
<summary>GetPrivateStaticClassBody()</summary>
<div markdown="1">
    
```cpp
/**
 * Helper template allocate and construct a UClass
 *
 * @param PackageName name of the package this class will be inside
 * @param Name of the class
 * @param ReturnClass reference to pointer to result. This must be PrivateStaticClass.
 * @param RegisterNativeFunc Native function registration function pointer.
 * @param InSize Size of the class
 * @param InAlignment Alignment of the class
 * @param InClassFlags Class flags
 * @param InClassCastFlags Class cast flags
 * @param InConfigName Class config name
 * @param InClassConstructor Class constructor function pointer
 * @param InClassVTableHelperCtorCaller Class constructor function for vtable pointer
 * @param InCppClassStaticFunctions Function pointers for the class's version of Unreal's reflected static functions
 * @param InSuperClassFn Super class function pointer
 * @param WithinClass Within class
 */
void GetPrivateStaticClassBody(
  const TCHAR* PackageName,
  const TCHAR* Name,
  UClass*& ReturnClass,
  void(*RegisterNativeFunc)(),
  uint32 InSize,
  uint32 InAlignment,
  EClassFlags InClassFlags,
  EClassCastFlags InClassCastFlags,
  const TCHAR* InConfigName,
  UClass::ClassConstructorType InClassConstructor,
  UClass::ClassVTableHelperCtorCallerType InClassVTableHelperCtorCaller,
  FUObjectCppClassStaticFunctions&& InCppClassStaticFunctions,
  UClass::StaticClassFunctionType InSuperClassFn,
  UClass::StaticClassFunctionType InWithinClassFn
  )
{
#if WITH_RELOAD
  if (IsReloadActive() && GetActiveReloadType() != EActiveReloadType::Reinstancing)
  {
  	UPackage* Package = FindPackage(NULL, PackageName);
  	if (Package)
  	{
  		ReturnClass = FindObject<UClass>((UObject *)Package, Name);
  		if (ReturnClass)
  		{
  			if (ReturnClass->HotReloadPrivateStaticClass(
  				InSize,
  				InClassFlags,
  				InClassCastFlags,
  				InConfigName,
  				InClassConstructor,
  				InClassVTableHelperCtorCaller,
  				FUObjectCppClassStaticFunctions(InCppClassStaticFunctions),
  				InSuperClassFn(),
  				InWithinClassFn()
  				))
  			{
  				// Register the class's native functions.
  				RegisterNativeFunc();
  			}
  			return;
  		}
  		else
  		{
  			UE_LOG(LogClass, Log, TEXT("Could not find existing class %s in package %s for reload, assuming new or modified class"), Name, PackageName);
  		}
  	}
  	else
  	{
  		UE_LOG(LogClass, Log, TEXT("Could not find existing package %s for reload of class %s, assuming a new package."), PackageName, Name);
  	}
  }
#endif
  
  ReturnClass = (UClass*)GUObjectAllocator.AllocateUObject(sizeof(UClass), alignof(UClass), true);
  ReturnClass = ::new (ReturnClass)
  	UClass
  	(
  	EC_StaticConstructor,
  	Name,
  	InSize,
  	InAlignment,
  	InClassFlags,
  	InClassCastFlags,
  	InConfigName,
  	EObjectFlags(RF_Public | RF_Standalone | RF_Transient | RF_MarkAsNative | RF_MarkAsRootSet),
  	InClassConstructor,
  	InClassVTableHelperCtorCaller,
  	MoveTemp(InCppClassStaticFunctions)
  	);
  check(ReturnClass);
  	
  InitializePrivateStaticClass(
  	InSuperClassFn(),
  	ReturnClass,
  	InWithinClassFn(),
  	PackageName,
  	Name
  	);
  
  // Register the class's native functions.
  RegisterNativeFunc();
}
```
</div>
</details> 

`GetPrivateStaticClassBody()` 함수는 **클래스 info 인스턴스**를 만들고, 의존 클래스(상속)들과 자기 자신 클래스를 `TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();`에 추가합니다. 또한, `GFirstPendingRegistrant`에 단방향 linked list로 연결됩니다.

그리고, 인자로 받은 RegisterNativeFunc 함수 포인터를 실행합니다. 

**RegisterNativeFunc**에 들어가는 함수는 UhtHeaderCodeGenerator가 generated.h와 gen.cpp에 생성하는 `StaticRegisterNatives클래스이름()` 함수입니다. (예를 들어, AActor 클래스의 경우 StaticRegisterNativesAActor() 입니다.)  

이 함수는 해당 클래스의 C++로 구현된 **Native UFunction**들을 등록하는 함수입니다.

```cpp
void UDSStatComponent::StaticRegisterNativesUDSStatComponent()
{
	UClass* Class = UDSStatComponent::StaticClass();
	static const FNameNativePtrPair Funcs[] = {
		{ "ApplyBuff", &UDSStatComponent::execApplyBuff },
		{ "GetDefaultStatByEnum", &UDSStatComponent::execGetDefaultStatByEnum },
		// ...
	};
	FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, UE_ARRAY_COUNT(Funcs));
}
```

모든 UFUNCTION 멤버 함수에 대해, ‘함수 이름’, ‘exec 함수 포인터’를 `NativeFunctionLookupTable`에 등록합니다.

<br>

결과 : 

- C++ Native 함수 포인터(exec 함수)를 `NativeFunctionLookupTable`에 등록
- `TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants` 에 클래스 등록에 필요한 정보 등록

---

## 4. OuterRegister

<details>
<summary>CoreUObject부터 등록합니다. (밑에 읽고 읽는 것을 추천합니다)</summary>
<div markdown="1">
    
우선, `FEngineLoop::AppInit()`에서 `FCoreDelegates::OnInit.Broadcast()` 해서 CoreUObject부터 등록한다.
    
`FCoreUObjectModule`의 `StartupModule()` 에서 `void UClassRegisterAllCompiledInClasses()` → `InnerRegister(Registrant)`가 호출된 후 `FCoreDelegates::OnInit.AddStatic(InitUObject);`
    
`InitUObject()`->`StaticUObjectInit()`→ `UObjectBaseInit()` → `UObjectProcessRegistrants()`
    
</div>
</details> 

`FEngineLoop::PreInitPostStartupScreen()` → `ProcessNewlyLoadedUObjects()`에서 `void UClassRegisterAllCompiledInClasses()` 가 끝나고

PendingRegistrants가 모두 없어질 때까지 해당 과정을 통해 리플렉션에 모두 등록합니다.

```cpp
bool bNewUObjects = false;
TArray<UClass*> AllNewClasses;
while (GFirstPendingRegistrant ||
  ClassRegistry.HasPendingRegistrations() ||
  StructRegistry.HasPendingRegistrations() ||
  EnumRegistry.HasPendingRegistrations())
{
  bNewUObjects = true;
  UObjectProcessRegistrants();
  UObjectLoadAllCompiledInStructs();
  
  FCoreUObjectDelegates::CompiledInUObjectsRegisteredDelegate.Broadcast(Package);
  
  UObjectLoadAllCompiledInDefaultProperties(AllNewClasses);
}
```

<br>

#### <u>UObjectProcessRegistrants()</u>> :

시스템에 클래스를 등록합니다.

`UObjectProcessRegistrants()` → `UObjectForceRegistration()` ->`DeferredRegister()`

`UObjectProcessRegistrants()`는 FPendingRegistrant* GFirstPendingRegistrant에 next 포인터로 연결되어 있는 FPendingRegistrant들을 등록합니다.

`DeferredRegister()`는 UObject의 `ClassPrivate`에 UClassStaticClass를 넣고, GUObjectArray에 등록하고 ClassMap에 등록합니다.  

#### <u>UObjectLoadAllCompiledInStructs()</u> :

Struct Registrations, Enum Registrations에 있는 것들을 등록합니다.  

#### <u>UObjectLoadAllCompiledInDefaultProperties()</u> :

**`DoPendingOuterRegistrations()`** 함수를 호출해서 Class Registrations에 있는 OuterRegister (Z_Construct_UClass_클래스이름) 함수 포인터를 호출합니다.

```cpp
UClass* Z_Construct_UClass_클래스이름()
{
  if (!Z_Registration_Info_UClass_클래스이름.OuterSingleton)
  {
    UECodeGen_Private::ConstructUClass(Z_Registration_Info_UClass_클래스이름.OuterSingleton, Z_Construct_UClass_클래스이름_Statics::ClassParams);
  }
  return Z_Registration_Info_UClass_클래스이름.OuterSingleton;
}
```

ConstructUClass()가 호출됩니다.

<details>
<summary>ConstructUClass() 코드 접기/펼치기</summary>
<div markdown="1">
    
```cpp
void ConstructUClass(UClass*& OutClass, const FClassParams& Params)
{
  if (OutClass && (OutClass->ClassFlags & CLASS_Constructed))
  {
  	return;
  }
  
  for (UObject* (*const *SingletonFunc)() = Params.DependencySingletonFuncArray, *(*const *SingletonFuncEnd)() = SingletonFunc + Params.NumDependencySingletons; SingletonFunc != SingletonFuncEnd; ++SingletonFunc)
  {
  	(*SingletonFunc)();
  }
  
  UClass* NewClass = Params.ClassNoRegisterFunc();
  OutClass = NewClass;
  
  if (NewClass->ClassFlags & CLASS_Constructed)
  {
  	return;
  }
  
  UObjectForceRegistration(NewClass);
  
  UClass* SuperClass = NewClass->GetSuperClass();
  if (SuperClass)
  {
  	NewClass->ClassFlags |= (SuperClass->ClassFlags & CLASS_Inherit);
  }
  
  NewClass->ClassFlags |= (EClassFlags)(Params.ClassFlags | CLASS_Constructed);
  // Make sure the reference token stream is empty since it will be reconstructed later on
  // This should not apply to intrinsic classes since they emit native references before AssembleReferenceTokenStream is called.
  if ((NewClass->ClassFlags & CLASS_Intrinsic) != CLASS_Intrinsic)
  {
  	check((NewClass->ClassFlags & CLASS_TokenStreamAssembled) != CLASS_TokenStreamAssembled);
  	NewClass->ReferenceSchema.Reset();
  }
  NewClass->CreateLinkAndAddChildFunctionsToMap(Params.FunctionLinkArray, Params.NumFunctions);
  
  ConstructFProperties(NewClass, Params.PropertyArray, Params.NumProperties);
  
  if (Params.ClassConfigNameUTF8)
  {
  	NewClass->ClassConfigName = FName(UTF8_TO_TCHAR(Params.ClassConfigNameUTF8));
  }
  
  NewClass->SetCppTypeInfoStatic(Params.CppClassInfo);
  
  if (int32 NumImplementedInterfaces = Params.NumImplementedInterfaces)
  {
  	NewClass->Interfaces.Reserve(NumImplementedInterfaces);
  	for (const FImplementedInterfaceParams* ImplementedInterface = Params.ImplementedInterfaceArray, *ImplementedInterfaceEnd = ImplementedInterface + NumImplementedInterfaces; ImplementedInterface != ImplementedInterfaceEnd; ++ImplementedInterface)
  	{
  		UClass* (*ClassFunc)() = ImplementedInterface->ClassFunc;
  		UClass* InterfaceClass = ClassFunc ? ClassFunc() : nullptr;
  
  		NewClass->Interfaces.Emplace(InterfaceClass, ImplementedInterface->Offset, ImplementedInterface->bImplementedByK2);
  	}
  }
  // ...
}
```
</div>
</details>  

- 위에 나왔던 `UObjectForceRegistration()` ->`DeferredRegister()` 이 호출됩니다. 자동 등록이 아니라 이후 등록에 대비해서인가?)
- 부모 클래스의 클래스 플래그 상속
- `CreateLinkAndAddChildFunctionsToMap()` : `Z_Construct_UFunction_`를 사용해서 멤버 함수들의 UFunction 객체를 생성(`ConstructUFunction`)하고, 이 객체들을 NextPtr(단방향 linked list)로 연결하고, 해당 UClass 안에 있는 FuncMap에 등록합니다.
- `ConstructFProperties`
- ImplementedInterface 정보 연결
- `StaticLink()`

모든 클래스 생성이 끝나면 CoreUObject → Engine → 그 외 순서로 생성이 끝났음을 NotifyEvent합니다.

```cpp
NotifyRegistrationEvent(PackageName, ClassName, ENotifyRegistrationType::NRT_Class, ENotifyRegistrationPhase::NRP_Finished, nullptr, false, Class);
```

그 후, CDO를 생성합니다.

`Class->GetDefaultObject()`

<br>

결과 :

- UClass, UFunction, UProperty 정적 객체 등록 및 생성 끝
- CDO 생성

---

# 그 외

### exec 함수(thunk 함수)

`NativeFunctionLookupTable`에 등록되는 함수입니다. 

**exec 함수 (thunk 함수)** : 네이티브 함수 핸들러입니다. 블루프린트, 네트워크 RPC에서 C++ 함수를 실행할 때 사용됩니다. 

execApplyBuff 함수 (파라미터 값들 읽어서 함수 호출)

```cpp
DEFINE_FUNCTION(UDSStatComponent::execApplyBuff)
{
  P_GET_ENUM(EDSStatType,Z_Param_InStatType);
  P_GET_ENUM(EOperationType,Z_Param_InOperationType);
  P_GET_PROPERTY(FFloatProperty,Z_Param_InBuffValue);
  P_GET_PROPERTY(FFloatProperty,Z_Param_InDuration);
  P_FINISH;
  P_NATIVE_BEGIN;
  P_THIS->ApplyBuff(EDSStatType(Z_Param_InStatType),EOperationType(Z_Param_InOperationType),Z_Param_InBuffValue,Z_Param_InDuration);
  P_NATIVE_END;
}
```

<details>
<summary>쓰인 매크로들</summary>
<div markdown="1">
    
```cpp
// ObjectMacros.h
// This macro is used to define a thunk function in autogenerated boilerplate code
#define DEFINE_FUNCTION(func) void func( UObject* Context, FFrame& Stack, RESULT_DECL )
    
// ScriptMacros.h
#define P_GET_ENUM(EnumType,ParamName)				EnumType ParamName = (EnumType)0; Stack.StepCompiledIn<FEnumProperty>(&ParamName);
    
#define P_GET_PROPERTY(PropertyType, ParamName)													\
    PropertyType::TCppType ParamName = PropertyType::GetDefaultPropertyValue();					\
    Stack.StepCompiledIn<PropertyType>(&ParamName);
    	
#define P_FINISH									Stack.Code += !!Stack.Code; /* increment the code ptr unless it is null */
    
#define P_NATIVE_BEGIN { SCOPED_SCRIPT_NATIVE_TIMER(ScopedNativeCallTimer);
#define P_NATIVE_END   }
    
#define P_THIS_OBJECT								(Context)
#define P_THIS_CAST(ClassType)						((ClassType*)P_THIS_OBJECT)
#define P_THIS										P_THIS_CAST(ThisClass)
```
</div>
</details> 
  
<details>
<summary>원본 코드</summary>
<div markdown="1">
    
```cpp
void UDSStatComponent::ApplyBuff(EDSStatType InStatType, EOperationType InOperationType, float InBuffValue, float InDuration)
{
  // 새로운 버프 항목 생성 및 추가
  FBuffEntry NewEntry;
  NewEntry.StatType = InStatType;
  NewEntry.OperationType = InOperationType;
  NewEntry.BuffID = NextBuffID++;
  NewEntry.BuffValue = InBuffValue;
  ActiveBuffs.Add(NewEntry);
  
  DS_LOG(DSStatLog, Log, TEXT("[%s] %s 버프 적용: ID=%d, %s %f, 지속시간: %f초"), *GetName(), *UEnum::GetValueAsString(InStatType), NewEntry.BuffID, *UEnum::GetValueAsString(InOperationType), InBuffValue, InDuration);
  
  // 버프 제거 타이머 설정
  FTimerHandle TimerHandle;
  FTimerDelegate TimerDelegate;
  TimerDelegate.BindUObject(this, &UDSStatComponent::RemoveBuff, NewEntry.BuffID);
  if (GetWorld())
  {
  	GetWorld()->GetTimerManager().SetTimer(TimerHandle, TimerDelegate, InDuration, false);
  	BuffTimerHandles.Add(NewEntry.BuffID, TimerHandle);
  }
  
  UpdateCurrentStat();
}
```
</div>
</details> 