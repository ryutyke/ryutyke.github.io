---
title: "[Algorithm] Behavior Tree"
excerpt: "게임 알고리즘"

categories:
  - Algorithm
tags:
  - [Algorithm, AI, Behavior, Tree]

permalink: /algorithm/behavior_tree/

toc: true
toc_sticky: true

date: 2024-02-04 03:00:00
last_modified_at: 2024-02-04
---
<br>

# 행동 트리 (Behavior Tree)

- 행동(Behavior)을 트리(tree) 자료구조로 기술.
- 한 덩어리의 태스크가 서브 트리(sub tree)를 이룬다.
- **깊이 우선 탐색**으로 순차적으로 행동 검사.

<br>

트리의 구성 요소
- root 노드 : 루트 노드.
- control flow 노드 : 루트도 리프(leaf)도 아닌 노드.
- execution 노드 : 리프 노드. 사실상 Behavior는 이것.

<br>

자식 노드는 부모 노드한테 결과를 반환합니다<br>
- Success (성공)
- Failure (실패)

<br>

## 노드

<div>
    <img src="/assets/images/posts_img/behavior_tree.png" alt="thumbnail" width="100%" min-width="700px" itemprop="image">
</div>
출처 : [https://steemit.com/ai/@leeyoosung/ai-behavior-tree](https://steemit.com/ai/@leeyoosung/ai-behavior-tree)

<br>

### Control flow 노드
(1) Composite Node<br>
하나 이상의 자식 노드를 가질 수 있다.<br>
- Sequence : 순차적으로 자식을 탐사하여 자식 노드에서 Failure가 반환되는 즉시 Failure를 반환. 모든 자식 노드가 Success되면 Success 반환.
- Selector : 순차적으로 자식을 탐사하여 자식 노드에서 Success가 반환되는 즉시 Success를 반환. 모든 자식 노드가 Failure되면 Failure 반환.


(2) Decorator Node<br>
오직 하나의 자식을 가지며 자식 노드의 결과를 변형하거나 반복하는 등의 역할을 하게 된다.<br>
- Condition : 조건에 따라 자식을 실행하거나 실패를 반환합니다.
- Loop : 일정 횟수, 혹은 무한히 자식을 반복 실행합니다.
- Inverter : 자식의 결과를 반대로 바꿔서 반환합니다.

<br>

### Execution 노드
최하위 노드로 행동 그 자체다. 실제 게임에 필요한 로직(걷기, 공격 등)이다.<br>

- Precondition(조건) : agent가 이 behavior를 실행할 수 있는가?
- Action : behavior 실행 시 agent가 실제로 하는 일

조건에 따라 Action을 하고, 부모 노드에게 Success나 Failure를 반환.

<br>

## 장점:
- Simplicity : AI 로직을 쉽게 파악 가능. 구현이 쉽다. 깊이 우선 탐색으로 진행됨.
- Stateless : 상태의 전환이 없으므로 이전 상태에 대해 기억할 필요가 없다.
- Unware of each other : 각 behavior는 다른 behavior를 몰라도 된다. 디커플링. FSM은 서로 얽히고 얽혀서 추가/삭제시 다른 애들도 고려했어야 함.
- Extensibility : 독립성이 좋아서 확장하기 좋음
- etc. : 노드의 재사용성 좋음, 트리 구조라 시각화 툴 제작이 용이하다.

## 단점
- 실행 시간이 FSM보다 크다.
- 잘못 구현하면 거대한 조건문을 양산하게 됨.
- 하드웨어 성능을 고려해야 함.
- Selector의 자식 노드가 많을수록 AI의 의사결정이 지연될 수 있다.
- 행동 트리의 탐색 순서가 행동에 영향을 미친다. (왼쪽 -> 오른쪽 탐색)

<br>

## TMI Zone
Utility Systems<br>
심즈 처럼 수 많은 가능한 행동이 있는 게임. 수 많은 **고려사항에 기반한 선호하는 행동의 선택.**<br>
이런 경우 Utility Systems => 확률을 넣는 방법도 있음. <br>
나중에 더 알아보자.

