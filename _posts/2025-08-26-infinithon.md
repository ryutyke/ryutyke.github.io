---
title: "[Plugin] 딴짓 방지 깜짝 퀴즈"
excerpt: "2025년 스마일게이트 해커톤인 인피니톤에서 구현한 플러그인입니다."

categories:
  - Portfolio
  - pfunrealengine
tags:
  - [Portfolio, Plugin, UnrealEngine]

permalink: /unrealengine/infinithon/

toc: true
toc_sticky: true

date: 2025-08-26 20:00:00
last_modified_at: 2025-08-26
---
<br>

<iframe width="560" height="315" src="https://www.youtube.com/embed/BZMotZ6Xuo0?si=pqQJsE8uHdTlGII5" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

- 영상 : [https://youtu.be/BZMotZ6Xuo0?si=kpoZ-AZSWlzLtPAb](https://youtu.be/BZMotZ6Xuo0?si=kpoZ-AZSWlzLtPAb)
- github 링크 : [https://github.com/ryutyke/AttentionPopup](https://github.com/ryutyke/AttentionPopup) 
 
<br>

스마일게이트 해커톤, 인피니톤에서 구현한 프로젝트입니다.  
언리얼 엔진에서 해당 플러그인을 추가하고 에디터 내 버튼을 통해 기능 ON/OFF가 가능합니다.

설정한 시간 동안 마우스, 키보드 입력이 없을 시 깜짝 퀴즈가 나옵니다.  
제한 시간 초과 및 오답 시 벌칙은 사용자가 구현하면 됩니다.  
데이터 테이블에서 다양한 형식의 퀴즈를 추가할 수 있습니다.  
언리얼 외 사용하는 개발 프로그램을 화이트리스트에 추가하면, 딴짓에서 제외됩니다.  

<br>

1박 2일이라는 짧은 시간에 다뤄보지 않은 내용들(에디터 UI, Window API 등)을 다루고, 협업하다 보니 많은 버그가 발생했습니다.  
특히, 대화형 인공지능에 의존하다보니 오류가 많았습니다.

강한 책임감과, 리더십, 다양한 프로젝트들을 진행하며 얻은 디버깅 실력과 코드 이해 능력, 협업 능력 덕분에 팀의 목표를 이룰 수 있었습니다.  
**특히 코드를 전부 이해할 시간이 없는 상황에서 저의 디버깅 능력은 빛을 발했습니다.**

지금까지 참여한 모든 해커톤과 게임잼에서 팀의 목표를 이뤘다는 점에서 강한 책임감, 협업 능력, 개발을 좋아하는 마음을 어필하고 싶습니다.

<br>

후속 지원 프로그램에 참여해 발전시켜 볼 생각입니다.
- 게임으로 빌드해 언리얼 엔진 개발자뿐만 아니라 모든 사람이 사용할 수 있게 만들기
- 벌칙 대신 펫 키우기 (집중 시간동안 경험치 상승, 퀴즈 시간 초과 시 경험치 감소)
- 코드 최적화
- 리팩토링