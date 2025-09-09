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

- 영상 : [https://youtu.be/BZMotZ6Xuo0?si=kpoZ-AZSWlzLtPAb](https://youtu.be/BZMotZ6Xuo0?si=kpoZ-AZSWlzLtPAb){:target="_blank"}
- github 링크 : [https://github.com/ryutyke/AttentionPopup](https://github.com/ryutyke/AttentionPopup){:target="_blank"} 
 
<br>

스마일게이트 해커톤, 인피니톤에서 구현한 프로젝트입니다.  

언리얼 엔진에서 해당 플러그인을 추가하고 에디터 내 버튼을 통해 기능 ON/OFF가 가능합니다.  

- 설정한 시간 동안 마우스, 키보드 입력이 없을 시 깜짝 퀴즈가 나옵니다.  
- 제한 시간 초과 및 오답 시 벌칙을 넣을 수 있습니다. (현재는 아두이노를 사용해 벌칙을 구현했습니다.)  
- 데이터 테이블에서 다양한 형식의 퀴즈를 추가할 수 있습니다.  
- 언리얼 외 사용하는 개발 프로그램을 화이트리스트에 추가하면, 딴짓에서 제외됩니다.  

<br>

1박 2일이라는 짧은 시간에 다뤄보지 않은 내용들(에디터 UI, Window API 등)을 다루고, 협업하다 보니 많은 버그가 발생했습니다.  
특히, 대화형 인공지능에 의존하다보니 오류가 많았습니다.  

그래도 소통을 통해 개인 작업 현황과 컨디션을 공유하고, 이를 고려해 작업을 재분배함으로써 효율적으로 일을 진행할 수 있었고, 결국 팀의 목표를 이룰 수 있었습니다.  
특히 코드를 전부 이해할 시간이 없는 상황에서 문제를 해결하는 **디버깅 능력**은 굉장히 중요했던 것 같습니다.  

다양한 해커톤과 게임잼 경험에서 느낀 건, 팀의 목표를 이루는 데에 가장 중요한 건 강한 책임감, 협업 능력, 그 일(개발)을 좋아하는 마음인 것 같습니다.    

<br>

후속 지원 프로그램에 참여해 발전시켜 볼 생각입니다.
- 게임으로 빌드해 언리얼 엔진 개발자뿐만 아니라 모든 사람이 사용할 수 있게 만들기
- 벌칙 대신 펫 키우기 (집중 시간동안 경험치 상승, 퀴즈 시간 초과 시 경험치 감소)
- 코드 최적화
- 리팩토링