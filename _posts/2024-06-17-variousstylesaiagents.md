---
title: "[RL] AI Agents of Various Gameplay Styles using RL"
excerpt: "2024년에 Unity 위에서 구현한 강화학습입니다."

categories:
  - Portfolio
  - pfunity
tags:
  - [Portfolio, ReinforcementLearning, Unity]

permalink: /Portfolio/Various_Gameplay_Styles_AI_Agents/

toc: true
toc_sticky: true

date: 2024-06-17 18:10:00
last_modified_at: 2024-06-17
---
<br>



<iframe width="560" height="315" src="https://www.youtube.com/embed/gqOXQiGKsC0?si=njdxzaoAIREO2OB4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

- 구현 영상 : [https://youtu.be/gqOXQiGKsC0?si=Lpg0p_-lMH9n96Aj](https://youtu.be/gqOXQiGKsC0?si=Lpg0p_-lMH9n96Aj)

## 주제
Implement AI Agents of Various Gameplay Styles using RL

## 참고한 논문
- CCP: Configurable Crowd Profiles

## 프로젝트 목표
게임 플레이어들의 게임플레이 스타일은 게임 요소(퀘스트 수행, 파티 플레이, 몬스터 사냥, 생존)에 대한 선호도에 따라 다양합니다. 실제 플레이어처럼 행동하는 다양한 플레이 스타일의 AI Agents를 구현하는 것이 프로젝트 목표입니다.

## 프로젝트 가치
해당 기술을 사용하면 실제 사람과 비슷하게 플레이 하는 다양한 스타일의 AI Agents를 적은 비용으로 만들 수 있습니다. 또한, 결과물을 분석하여 다양한 게임플레이 방식이 수용될 수 있게 밸런스를 맞추는 자료로 활용할 수 있습니다.

## 결론 및 제언
생성된 다양한 Agents들은 랜덤하게 정해진 선호도에 맞게 행동합니다. 즉, 다양한 플레이 스타일에 맞게 서로 다른 행동을 수행하는 AI Agents들을 강화학습을 통해 쉽게 구현할 수 있었습니다.<br>
학습 과정에서 행동 간 밸런스를 맞춰주기 위해 보상 값을 수정했는데, 이와 같은 방법으로 게임 요소간의 밸런스를 조절할 수 있을 것 같습니다.<br>
해당 프로젝트에서 사용한 게임은 Agent가 할 수 있는 행동의 종류가 적고, 메시 등을 제외하고는 시각화를 하지 않았기에 나타나는 스타일 종류가 적고 분석에 시간이 많이 들 수도 있지만, 스킬이나 채집 등 좀 더 다양한 동작이 가능하고 이펙트 등의 시각화가 잘 되어 있는 게임의 경우 해당 기술 사용 시의 이점이 더 커질 것이라 생각합니다.
