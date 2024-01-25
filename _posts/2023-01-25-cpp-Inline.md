---
title: "[C/C++] inline 함수"
excerpt: "함수 호출 대신 함수의 내용을 호출한 부분에 직접 삽입하여 실행하도록 컴파일러에게 제안"

categories:
  - C/C++
tags:
  - [C, C++, inline]

permalink: /c-cpp/inline/

toc: true
toc_sticky: true

date: 2024-01-25 15:12:00
last_modified_at: 2024-01-25
---
<br>

## inline

>함수 호출 대신 함수의 내용을 호출한 부분에 직접 삽입하여 실행하도록 컴파일러에게 제안하는 것.

<br>

### 장점 : 
- 함수 호출과 관련된 오버헤드(반환 주소, 인수, 함수로 이동, 리턴 처리)를 제거해서 실행 속도 빨라진다.
- **컴파일러가 인라인 처리 된 코드로 최적화를 할 수 있게 만든다.**

<br>

### 단점 : 
- 컴파일 시간이 늘어나고 바이너리 파일(실행 파일)의 용량이 커진다.

<br>

### 그 외 :
- inline 함수는 컴파일러에게 하는 **제안이지 강제 명령이 아니다.** 
따라서 inline함수로 선언된 부분을 일반 함수로 처리하기도 한다. (ex. 재귀 함수처럼 복잡한 함수, 반복문을 포함하는 함수 등)
또한, 컴파일러가 inline함수를 직접 선언하여 컴파일 하기도 한다.

- 작은 규모의 함수나 빈번하게 호출되는 함수에 사용하기 좋다. (ex. 접근자 함수, 간단한 계산 함수)

'''cpp
inline int max(int a, int b)
{
    return a < b ? b : a;
}
'''

- 클래스 정의 내에 함수 정의를 배치하면 암시적으로 인라인 함수로 간주된다.

'''cpp
#include <iostream>

class MyClass
{
public:
    void print() { std::cout << i; }   // Implicitly inline

private:
    int i;
};
'''

- 템플릿(Template) 함수에 사용 가능하다.

<br>

참고 사이트 :
https://learn.microsoft.com/ko-kr/cpp/cpp/inline-functions-cpp?view=msvc-170