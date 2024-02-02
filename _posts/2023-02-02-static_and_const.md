---
title: "[C/C++] Static과 Const"
excerpt: "코드 실행 결과로 알아보는 Static과 Const"

categories:
  - C/C++
tags:
  - [C, C++, static, const]

permalink: /c-cpp/static_and_const/

toc: true
toc_sticky: true

date: 2024-02-02 17:30:00
last_modified_at: 2024-02-02
---
<br>

## Static

```cpp
#include<iostream>
using namespace std;

void foo()
{
	static int a = 5; // 함수 여러 번 실행하면 처음 한 번만 실행됨.
	a++;
	cout << a << endl;
}

int main()
{
	foo(); // 6
	foo(); // 7

	static int a = 5;
	cout << a << endl; // 5
	foo(); // 8

	// static int a = 9; 여기서 여러 번 하면 재정의 컴파일 에러

	return 0;
}
```

<br>

## Const

```cpp 
#include<iostream>
using namespace std;

const int a = 1;

void foo()
{
	cout << "foo: " << a << endl;
}

int main()
{
	foo(); // 1
	
	// const int a = 4; 하고 
	// foo() 해도 결과는 1

	cout << a << endl; // 1
	int a = 5;
	cout << a << endl; // 5

	const int b = 2;
	// int b = 3; 같은 곳에선 안 됨. 재정의 오류
	// b = 3; const 값 변경 안 됨.
	cout << b << endl; // 2

	return 0;
}
```
