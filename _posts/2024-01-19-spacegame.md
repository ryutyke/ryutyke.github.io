---
title: "[Game] Space Game"
excerpt: "2019년에 Pygame으로 만든 게임입니다."

categories:
  - Portfolio
tags:
  - [Portfolio, Game, Pygame]

permalink: /Portfolio/SpaceGame/

toc: true
toc_sticky: true

date: 2024-01-19
last_modified_at: 2024-01-20
---
<br>

게임 영상 : [https://www.youtube.com/watch?v=Al0VxvH8jYo](https://www.youtube.com/watch?v=Al0VxvH8jYo)

### 1. 스토리

초록색은 내 친구입니다. 근데 빨간색이 내 친구를 죽입니다. 그래서 빨간색에게 복수하는 내용입니다.
스토리는 게임 시작 전 사진으로 설명을 합니다.
또한 게임이름, 배경음악, 그리고 검정배경으로 우주에서 적을 물리치는 게임이라는 것을 표현했습니다.

### 2. 규칙

처음 인트로 화면과 인트로 화면에서 들어갈 수 있는 Rule 화면에 설명했습니다.
조작키는 wasd (이동), 좌우 방향키(총구 돌리기), 스페이스바(총알 발사) 입니다.
조작키는 키보드의 모양을 고려하여 편하게 잡을 수 있게 만들었고, 양손을 풍족하게 사용할 수 있게 했습니다.

게임 속 초록색과 빨간색은 우주 비행선입니다. 따라서 **둘 다 플레이어와 부딪히면 게임오버**입니다.
**초록색은 팀이라 죽이면 게임오버, 빨간색은 적이라 죽이면 점수가 올라갑니다.**

Easymode와 Hardmode를 만들었으며, Easymode는 Hardmode에 비해 플레이어의 속도가 느려 보다 정확하게 다른 우주선을 피할 수 있습니다.
또한, 시간당 나오는 우주선의 수를 줄였고, 빨간색 우주선의 비율을 늘여 총알 발사시 초록색을 맞혀 게임오버 되는 경우를 줄이고, 높은 점수를 받기 쉽습니다.

### 3. 배경음악, 효과음

Free sound라는 홈페이지에서 퍼플릭도메인인 음원을 다운하고, Audacity로 편집해서 사용했습니다.
배경음악: 메인메뉴, 게임화면
효과음: 총알발사, 빨간색우주선 격추, 게임오버

### 4. 그림

그림판으로 그린 후, Gimp를 통해 색상 편집, 빈공간 투명화를 진행했습니다.

## 스토리 화면

<div>
    <img src="/assets/images/spacegame/1.png" alt="Intro Screen1" width="70%" min-width="700px" itemprop="image">
</div>

<div>
    <img src="/assets/images/spacegame/2.png" alt="Intro Screen2" width="70%" min-width="700px" itemprop="image">
</div>

<div>
    <img src="/assets/images/spacegame/3.png" alt="Intro Screen3" width="70%" min-width="700px" itemprop="image">
</div>

## 메인메뉴 화면

<div>
    <img src="/assets/images/spacegame/4.png" alt="Main Screen" width="70%" min-width="700px" itemprop="image">
</div>

## 규칙 소개 화면

<div>
    <img src="/assets/images/spacegame/5.png" alt="Rule Screen" width="70%" min-width="700px" itemprop="image">
</div>

## 이지모드 화면

<div>
    <img src="/assets/images/spacegame/6.png" alt="Easy Mode" width="70%" min-width="700px" itemprop="image">
</div>

우주선 개수가 적으며, 빨간 우주선 비율이 큼.

## 하드모드 화면

<div>
    <img src="/assets/images/spacegame/7.png" alt="Hard Mode" width="70%" min-width="700px" itemprop="image">
</div>

우주선 개수가 많으며, 빨간 우주선 비율이 작음.

## 게임오버 화면

<div>
    <img src="/assets/images/spacegame/8.png" alt="Gameover Screen" width="70%" min-width="700px" itemprop="image">
</div>


클래스를 통해 우주선들(플레이어, 초록색, 빨간색), 총알을 구현했습니다.
플레이어 총구 방향을 조절할 때, 플레이어의 크기가 변하게 해서, BGM과 시너지 효과를 냈습니다. 

친구들이 재밌게 플레이 했습니다.
