---
title: "[C/C++] C++ 언어 특징"
excerpt: "Features of C++"

categories:
  - C/C++
tags:
  - [C, C++]

permalink: /c-cpp/features_of_cpp/

toc: true
toc_sticky: true

date: 2024-02-02 16:00:00
last_modified_at: 2024-02-02
---
<br>


## C++ 언어 특징
-	C 언어로 작성된 프로그램과의 호환성 유지
-	[**객체 지향 프로그래밍**](https://ryutyke.github.io/c-cpp/object-oriented_programming/)의 개념을 도입 (캡슐화 + 정보은닉, 추상화, 상속, 다형성)
-	타입 체크를 엄격히 하여 런타임 오류의 가능성을 줄이고 디버깅을 도움.
-	실행 시간의 효율성 저하를 최소화 [**인라인 함수**](https://ryutyke.github.io/c-cpp/inline/)

## C++에 추가된 기능
-	(연산자) 오버로딩 : 매개 변수의 개수나 타입이 서로 다른 동일한 이름의 함수 선언 가능
-	디폴트 매개 변수 : 매개 변수에 값이 전달되지 않는 경우 디폴트 값 전달되도록 함수 선언 가능
-	참조 : 변수에 별명을 붙여 변수 공간을 같이 사용할 수 있는 참조의 개념을 도입
-	참조에 의한 호출 : 함수 호출시 참조를 전달할 수 있다.

```cpp
#include<iostream>
using namespace std;

void foo(int* a)
{
	cout << *a << endl;
}

void foo2(int& a)
{
	cout << a << endl;
}

int main()
{
	int b = 1;
	int& c = b;
	int* d = &c;
	
	foo(&b);
	foo2(b);

	cout << endl;

	*d = *d + 1;
	cout << b << " " << c << endl; // b = 2, c = 2

	c++;
	cout << b << " " << *d << endl; // b = 3, d = 3

	return 0;
}
```

<br>

-	new와 delete 연산자 : 동적 메모리 할당, 해제를 위한 두 연산자 도입 <br>
(malloc, free와의 차이점 : 생성자, 소멸자가 호출된다. 연산자라 오버로딩 가능하다. 크기를 지정해주지 않아도 된다. 생성 에러 관련 예외 처리가 되어있다.<br>
realloc 같은 크기 재할당 기능이 없어서 메모리를 새로 할당하고, 데이터를 새로 할당 받은 곳에 복사하고, 원래 메모리를 해제하는 과정을 직접 해야 한다.)

-	제네릭 함수와 클래스 : 함수나 클래스를 데이터 타입에 의존하지 않고 일반화 시킬 수 있다. (template)
-	[**인라인 함수**](https://ryutyke.github.io/c-cpp/inline/) : 자주 호출되는 함수의 경우 호출 대신 함수 코드를 확장 삽입한다. 실행 시간을 단축할 수 있다.
