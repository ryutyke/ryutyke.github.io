---
title: "[RL] Learning to Get Up"
excerpt: "2023년에 Unreal Engine 위에서 구현한 강화학습"

categories:
  - Portfolio
tags:
  - [Portfolio, ReinforceLearning, UnrealEngine]

permalink: /Portfolio/LearningToGetUp/

toc: true
toc_sticky: true

date: 2024-01-20 02:10:00
last_modified_at: 2024-01-20
---
<br>

구현 영상 : [https://www.youtube.com/watch?v=sk4Bt2QC1OI&t=1s](https://www.youtube.com/watch?v=sk4Bt2QC1OI&t=1s)

## 주제
SAC 강화학습 알고리즘, Strong to Weak 방법으로 기립 애니메이션 생성

## 참고한 논문
Tao, T., Wilson, M., Gou, R., & van de Panne, M. (2022, August 7). Learning to Get Up. Special Interest Group on Computer Graphics and Interactive Techniques Conference Proceedings. ACM. 

## 설명

### 동기 
게임 속 캐릭터의 기립 모션이 부자연스러웠다. 누워있는 모습의 형태는 다양하지만, 모든 초기 상태에 대해서 애니메이션을 제작할 수 없기 때문이다. 강화학습으로 어떤 초기 상태에서라도 일어날 수 있는 액터를 학습할 수 있다면 다양한 초기 상태에 대해서 아주 쉽게 애니메이션을 제작할 수 있을 것이다.

### 내용 
참고 논문에서 mujoco 엔진을 사용했는데, 게임 제작에 많이 쓰이는 언리얼 엔진에서 기본 캐릭터로 학습.

### 결과 
어떻게 누워있든지 상관없이 일어날 수 있는 결과물이 나왔다. 결과물이 후처리가 필요한 퀄리티이긴 하지만, 더 긴 시간 학습시키거나 더 적절한 파라미터 값과 리워드를 사용한다면 더 좋은 결과물이 나올 것이다. 또한 참고한 논문에서 소개한 slow까지 적용한다면 자연스러운 모션이 나올 것이다.

### 해당 기술의 장점
1. 학습 훈련 데이터 (기존 애니메이션, 영상)이 필요 없다.
2. 모션 캡쳐로는 쉽지 않은 다리 여러 개, 팔 여러 개, 긴 팔 등 인간이 아닌 비현실 개체에 대해서도 애니메이션 생성이 가능하다.

기립 애니메이션 외에도 다양한 동작을 Reward 값의 변경만으로 학습시킬 수 있었다. SAC와 Strong to Weak를 사용하여 학습 데이터 없이 빠르게 학습시킬 수 있었다.