---
title: "[C/C++] 메모리 구조"
excerpt: "Memory structure"

categories:
  - C/C++
tags:
  - [C, C++, heap, stack]

permalink: /c-cpp/memory_structure/

toc: true
toc_sticky: true

date: 2024-02-02 17:20:00
last_modified_at: 2024-02-02
---
<br>

## Memory Structure

<div>
    <img src="/assets/images/posts_img/memory_structure.png" alt="memory_structure" width="100%" min-width="300px" itemprop="image">
</div>

- Stack section : Function parameters, Return address, Local variables (컴파일 타임)
- Heap section : 동적 할당된 메모리 (런 타임)
- Data section : global and static variables (initialized, uninitialized, read-only 세 가지로 나눌 수 있음, 컴파일 타임)
- Text section : program code (read only, 컴파일 타임)

Stack은 높은 주소에서 낮은 주소로<br>
Heap은 낮은 주소에서 높은 주소로

<br>

## TMI Zone
const 변수는 Data section에 있는 read-only로 들어가는가? : X<br>
컴파일러에 따라서 다르며, 그냥 코드에 직접 대입하는 경우도 있고. 함수 내부에 넣으면 Stack에 들어가기도 하고.
