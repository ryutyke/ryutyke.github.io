---
title: "UE5 Main Loop (작성 중)"
excerpt: "언리얼 엔진의 main()은 어딜까"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Loop]

permalink: /unrealengine/loop/

toc: true
toc_sticky: true

date: 2025-03-25 20:00:00
last_modified_at: 2025-03-25
---
<br>

[\Engine\Source\Runtime\Launch\Private\Windows\LaunchWindows.cpp] 에 있는 WinMain()에서 시작합니다.  

WinMain() ⇒ LaunchWindowsStartup() ⇒ GuardedMain()  

[Launch.cpp]  
GEngineLoop는 FEngineLoop 클래스의 객체입니다. FEngineLoop 클래스는 LaunchEngineLoop.h 파일에 있고, PreInit(), Init(), Tick() 등 엔진 게임 루프가 정의되어 있습니다.

## GuardedMain()

- FCoreDelegates::GetPreMainInitDelegate().Broadcast();
- PreInit
    - EnginePreInit() ⇒ GEngineLoop.PreInit()
        - PreInitPreStartupScreen()
        - PreInitPostStartupScreen()
- Init
    - if Editor :  EditorInit(GEngineLoop) ⇒ (**IEngineLoop**)GEngineLoop.Init()
    - else : EngineInit() ⇒ GEngineLoop.Init()
- Tick
    - EngineTick() ⇒ GEngineLoop.Tick();
- Exit
    - if Editor : EditorExit()
    - else : return
    - 소멸자 ⇒ EngineExit() ⇒ GEngineLoop.Exit();

## PreInit

필요한 모든 엔진, 프로젝트 및 플러그인 모듈을 로드. 모듈이 메모리에 로드될 때 **정적 변수(static variables)**를 통해 리플렉션 등록 함수가 자동으로 호출.

- PreInitPreStartupScreen()
    - LoadCoreModules() : CoreUObject 로드
        - CoreUObject의 InnerRegister
        
        ```cpp
        FModuleManager::Get().LoadModule(TEXT("CoreUObject"))
        ```
        
    - LoadPreInitModules() : 필수 모듈 로드
        
        ```cpp
        FModuleManager::Get().LoadModule(TEXT("Engine"));
        FModuleManager::Get().LoadModule(TEXT("Renderer"));
        FModuleManager::Get().LoadModule(TEXT("AnimGraphRuntime"));
        FPlatformApplicationMisc::LoadPreInitModules();
        // ...
        ```
        
        - FPlatformApplicationMisc : FGenericPlatformApplicationMisc은 크로스 플랫폼을 지원하기 위해 운영체제의 주요 기능들을 인터페이스화 해둔 것이다. 윈도우에서는 이를 상속 받은 FWindowsPlatformApplicationMisc가 사용되며, FPlatformApplicationMisc로 typedef한 것이다.
    - AppInit()
        - ELoadingPhase::EarliestPossible
            - ELoadingPhase : 프로젝트나 플러그인 모듈 생성 시, 해당 모듈이 로드되는 시점을 지정할 수 있다.
        - ELoadingPhase::PostConfigInit
        - FCoreDelegates::OnInit.Broadcast();
            - CoreUObject의 UObjectProcessRegistrants()
    - ELoadingPhase::PostSplashScreen
- PreInitPostStartupScreen()
    - ELoadingPhase::PreEarlyLoadingScreen
    - ProcessNewlyLoadedUObjects() : 모든 클래스들 InnerRegister, OuterRegister
    - ELoadingPhase::PreLoadingScreen
    - FShaderCodeLibrary::OpenLibrary
    - FEngineLoop::LoadStartupCoreModules
        
        ```cpp
        FModuleManager::Get().LoadModule(TEXT("Core"));
        FModuleManager::Get().LoadModule(TEXT("Networking"));
        // ...
        ```
        
    - FEngineLoop::LoadStartupModules()
        - ELoadingPhase::PreDefault
        - ELoadingPhase::Default
        - ELoadingPhase::PostDefault

<br>

## Init

// 이어서 작성



---

### LoadModule() 과정

`FModuleManager::LoadModuleWithFailureReason()`

`FindModule(InModuleName)`

`AddModule(InModuleName)`

`ModuleInitializer.Execute()`

`ModuleInfo->Module->StartupModule();` 

`ModulesChangedEvent.Broadcast(InModuleName, EModuleChangeReason::ModuleLoaded);`