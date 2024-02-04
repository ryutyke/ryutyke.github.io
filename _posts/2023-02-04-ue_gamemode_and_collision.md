---
title: "[UnrealEngine] GameMode and Collision"
excerpt: "[임시]언리얼 엔진이 게임 제작을 위해 제공하는 프레임워크"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Gameplay, Framework, Gamemode, Collision]

permalink: /unrealengine/gamemode_and_collision/

toc: true
toc_sticky: true

date: 2024-02-04 22:00:00
last_modified_at: 2024-02-04
---
<br>

>코딩 테스트 공부하면서 막 기록한 내용... 조만간 정리 예정

## 게임모드와 게임 스테이트
Game Mode (게임 모드)와 Game State(게임 스테이트)
게임 모드 : 규칙
게임 스테이트 : 상태

<br>

### 게임모드
플레이하는 데 필요한 플레이어 수, 그 플레이어가 게임에 참가하는 방식 등의 기본적인 게임 규칙을 게임 모드에 구현.<br> 
한 번에 하나의 게임 모드만 사용할 수 있다.<br>
게임 모드에 쓰이는 베이스 클래스는 AGameModeBase.<br> 
게임 시작, 로그인, 스폰 위치 등과 관련된 기본 함수들이 있고,

<br>

## 콜리전
- 스태틱메시 애셋 : 스태틱메시 애셋에 콜리전 영역을 심는 방법
- 기본 도형 컴포넌트 : 구체, 박스, 캡슐의 기본 도형을 사용해 충돌 영역을 지정하는 방법
- 피직스 애셋 : 캐릭터의 각 부위에 충돌영역을 설정하는 방법, 래그돌 효과를 구현할 때 사용

<br>

### 콜리전 채널
- WorldStatic 움직이지 않는 정적인 배경액터에 사용하는 콜리전 채널, 주로 스태틱 메시에 있는 스태틱 메시 컴포넌트에 사용
- WorldDynamic 움직이는 액터에 사용하는 콜리전 채널, 블루 프린트에 속한 스태틱 메시 컴포넌트에 사용
- Pawn
- etc.

<br>

### Collision Enabled 항목 : 해당 컴포넌트에서 물리 기능을 어떻게 사용할지 지정하는 항목
- NoCollision : 물리적으로 아무일도 일어나지 않는다.
- Query : 두 물체의 충돌 영역이 서로 겹치는지 테스트하는 설정
- Physics : 물리적인 시뮬레이션을 사용할 때 설정

<br>

### 상대 컴포넌트의 콜리전 채널과 어떻게 반응할지 지정하는 항목이 있다.
- 무시(Ignore) : 콜리전이 있어도 아무 충돌이 일어나지 않는다.
- 겹침(Overlap) : 물체가 뚫고 지나갈 수 있지만 이벤트가 발생한다.
- 블록(Block) : 물체가 뚫고 지나가지 못하도록 막는다.
