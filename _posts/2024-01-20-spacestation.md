---
title: "[Game] Space Station"
excerpt: "2023년에 Unreal Engine으로 만든 게임입니다. 실시간 보로노이 파쇄 기술을 적용했습니다."

categories:
  - Portfolio
tags:
  - [Portfolio, Game, UnrealEngine]

permalink: /Portfolio/SpaceStation/

toc: true
toc_sticky: true

date: 2024-01-20 14:50:00
last_modified_at: 2024-01-20
---
<br>

<!--
<div>
    <img src="/assets/images/thumbnail/spacestation.png" alt="thumbnail" width="100%" min-width="700px" itemprop="image">
</div>
-->

<iframe width="560" height="315" src="https://www.youtube.com/embed/vEbmR1M-XIA?si=VdZ34_Grih7Rrb8O" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

- 게임 영상 : [https://www.youtube.com/watch?v=vEbmR1M-XIA](https://www.youtube.com/watch?v=vEbmR1M-XIA)
- github 링크 : [https://github.com/ryutyke/SpaceStation](https://github.com/ryutyke/SpaceStation)

<br>

[(관련 포스팅) Real-time Voronoi Fracturing](https://ryutyke.github.io/Portfolio/Real-time_Voronoi_Fracturing/)

총알 충돌 지점 근처 점들을 사용한 Real-time Voronoi Fracturing 기술을 사용하여 만든 게임 Space Station입니다.

## 게임의 장르 
FPS

## 게임의 배경 
Space Station에서 플레이어는 밀폐된 통로들에서 나아갈 수 있는 방향으로만 자동적으로 움직인다. 밀폐된 통로는 플레이어의 이동방향에서 상, 하, 좌, 우, 전진(혹은 후진) 방향으로 랜덤 생성되며, 통로 안에는 플레이어를 Game Over시킬 수 있는 적색, 녹색, 청색의 투명한 유리 개체가 랜덤한 크기와 간격으로 생성된다. 좌측 상단엔 현재 점수, 현재 스테이지, 사용 가능한 총알 수 UI가 있으며, 우측 하단엔 게이지 형태의 UI가 있다.

## 플레이어의 조작 
플레이어는 마우스를 움직여 시야를 조정할 수 있고, 왼쪽 마우스 클릭을 통해 총알을 발사한다.

## 게임의 규칙 
플레이어가 유리 개체에 부딪히면 게임 오버 된다. 게임 오버 되지 않으려면 총알을 발사해서 유리를 깨트려야 한다. 총알의 색과 유리의 색이 같아야 유리가 깨지며, 유리를 깨트릴 때마다 점수가 1점씩 오른다. 총알의 색은 디스플레이에 우측 하단에 위치한 게이지를 가리키는 화살표로 확인할 수 있다. UI 게이지에서 화살표가 가리키는 색상이 발사할 때의 총알의 색상이 된다. 화살표는 게이지 내에서 좌우로 움직이고 플레이어의 생존 시간이 지날수록 속도가 빨라진다. 플레이어의 이동속도도 점점 빨라진다. 오래 생존하면서 최대한 많은 유리를 깨트려야 높은 점수를 얻을 수 있다.