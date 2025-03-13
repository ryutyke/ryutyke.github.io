---
title: "형변환과 RTTI"
excerpt: "업캐스팅 -> 다운캐스팅 -> dynamic_cast -> RTTI"

categories:
  - C/C++
tags:
  - [C, C++]

permalink: /c-cpp/rtti/

toc: true
toc_sticky: true

date: 2025-03-13 16:00:00
last_modified_at: 2024-03-13
---
<br>

## 다음 내용들을 다룹니다.
- Upcasting과 Downcasting  
- dynamic_cast  
- RTTI (Run-Time Type Information)  

<br>

---
## Upcasting, Downcasting

<br>

### 업캐스팅(Upcasting)
업캐스팅은 파생 클래스(derived class)의 객체를 기본 클래스(base class)의 포인터나 참조로 사용하는 것입니다.  
파생 클래스 객체를 기본 클래스 포인터에 할당하면, 파생 클래스의 고유 멤버는 사용할 수 없고, 기본 클래스에 정의된 멤버만 접근할 수 있습니다.  

특징 :  

- **안전성** : 업캐스팅은 항상 안전합니다. (자식 클래스는 부모 클래스의 모든 멤버를 포함하므로)  
- **암시적 변환** : 별도의 명시적 캐스트 없이 자동으로 변환됩니다.  
- **다형성 활용** : 여러 파생 클래스를 기본 클래스 타입으로 관리할 수 있어, 하나의 인터페이스로 다양한 객체를 제어할 수 있습니다.  
- **가상 소멸자 (Virtual Destructor)** : 업캐스팅한 객체를 기본 클래스 포인터로 delete할 때, 기본 클래스의 소멸자가 virtual이어야 파생 클래스의 소멸자가 올바르게 호출됩니다. 그렇지 않으면 메모리 누수나 예기치 않은 동작이 발생할 수 있습니다.  

```cpp
class Base
{
public:
	virtual ~Base() {}
	virtual void show() { std::cout << "Base" << std::endl; }
};

class Derived : public Base
{
public:
	void show() override { std::cout << "Derived" << std::endl; }
	void derivedFunc() { std::cout << "Derived function" << std::endl; }
};

int main()
{
	Derived d;
	Base* pBase = &d;  // 업캐스팅: Derived -> Base (암시적 변환)
	pBase->show();     // 다형성에 의해 Derived의 show() 호출
	// pBase->derivedFunc(); // 오류: Base에는 derivedFunc가 없음
	return 0;
}
```

<br>

### 다운캐스팅(Downcasting)

다운캐스팅은 기본 클래스의 포인터나 참조를 다시 파생 클래스의 포인터나 참조로 변환하는 것입니다.  
다운캐스팅을 통해 파생 클래스의 고유 멤버(예, derivedFunc())에 접근할 수 있습니다.  

특징 :  

- **명시적 변환 필요** : 다운캐스팅은 암시적으로 이루어지지 않으므로, 명시적으로 캐스팅을 해주어야 합니다.  
- **안정성 문제** : 실제 객체가 변환하려는 파생 클래스가 아니라면, 잘못된 캐스팅이 발생할 수 있습니다. 예를 들어, C 스타일 캐스팅의 경우에는 컴파일러가 강제로 타입 변환을 수행하므로 정의되지 않은 동작을 초래할 수 있습니다.  
    - **dynamic_cast** : 다형성이 적용된 (즉, 최소한 하나 이상의 virtual 함수가 있는) 기본 클래스의 경우, dynamic_cast를 사용하면 RTTI를 사용해서 **런타임에 타입 체크**가 이루어져 안전하게 다운캐스팅할 수 있습니다. 만약 실제 객체가 변환하려는 파생 클래스가 아니라면, 런타임 에러(nullptr 반환 또는 bad_cast 예외)가 발생합니다.  

```cpp
class Base
{
public:
	virtual ~Base() {}
};

class Derived : public Base
{
public:
	void derivedFunc() { std::cout << "Derived function" << std::endl; }
};

int main()
{
	Base* pBase = new Derived;  // 업캐스팅은 안전
	// Derived* pDerived = (Derived*)pBase; // C 스타일 캐스팅: 위험할 수 있음

	// 안전한 다운캐스팅: dynamic_cast 사용
	Derived* pDerived = dynamic_cast<Derived*>(pBase);
	if (pDerived)
	{
		pDerived->derivedFunc();  // Derived 클래스 고유 함수 호출 가능
	}
	else
	{
		std::cout << "다운캐스팅 실패" << std::endl;
	}
	return 0;
}
```

<br>

danamic_cast 시에 만약 파생 클래스가 아니라면,  

- **포인터 변환의 경우:** 잘못된 캐스팅으로 인해 nullptr을 반환합니다.  
- **참조 변환의 경우:** std::bad_cast 예외를 발생시킵니다.  

```cpp
class Base
{
public:
	virtual ~Base() {}
};

class Derived : public Base
{
};

int main()
{
	Base* pBase = new Base();
	Derived* pDerived = dynamic_cast<Derived*>(pBase); // nullptr 반환
	if (nullptr == pDerived)
	{
		std::cout << "nullptr" << std::endl;
	}

	try
	{
		Base b;
		Derived pDerived1 = dynamic_cast<Derived&>(b);
	}
	catch (std::bad_cast& e)
	{
		std::cout << "Exception : " << e.what(); // "Bad dynamic_cast!"
	}

	return 0;
}
```

<br>

---
## 런타임 타입 정보(RTTI)

dynamic_cast는 런타임에 실제 객체의 타입 정보를 확인하기 위해 RTTI를 사용합니다.  

[Microsoft Learn : RTTI](https://learn.microsoft.com/ko-kr/cpp/cpp/run-time-type-information?view=msvc-170)

**RTTI(Run-Time Type Information)** : 프로그램 실행 중에 객체의 타입이 결정될 수 있도록 하는 메커니즘입니다. 가상 함수 테이블에 있는 type_info 객체에 대한 포인터를 사용합니다.  
- typeid 연산자 : 객체의 정확한 타입을 식별하는 데 사용됩니다.  
- type_info 클래스 : 연산자가 반환한 타입 정보(typeid)를 보관하는 데 사용됩니다.  

<br>

### typeid 연산자
\<typeinfo> 헤더에 존재하는 typeid 연산자를 통해 데이터의 타입을 얻어올 수 있습니다.  

- typeid(변수)  
- typeid(데이터 타입)  

반환 타입은 const std::type_info& 입니다.  

<br>

### type_info 클래스

[Microsoft Learn : type_info 클래스](https://learn.microsoft.com/ko-kr/cpp/cpp/type-info-class?view=msvc-170)

type_info는 typeid로 얻어온 데이터 타입을 보관하는 클래스입니다.  

- type_info 클래스의 객체를 생성하는 유일한 방법은 typeid 연산자의 반환값을 사용하는 것입니다. (상수 객체)  
- 복사 생성자, 복사 대입 연산자는 삭제되어 객체를 복사 생성하거나 할당할 수 없습니다.  
- name()이라는 멤버 함수를 통해서 타입의 이름을 얻을 수 있습니다.  

```cpp
#include<iostream>
#include<typeinfo>
using namespace std;

int main()
{
	int num = 10;
	auto str = "ABC";

	const std::type_info& info1 = typeid(num);
	const std::type_info& info2 = typeid(str);

	cout << info1.name() << endl;
	cout << info2.name() << endl;

	return 0;
}
```

결과 :  
<div>
    <img src="/assets/images/rtti/result1.png" alt="matrix" width="70%" min-width="300px" itemprop="image">
</div>

<br>

이렇게 활용할 수 있습니다.  
```cpp
#include<iostream>
#include<typeinfo>
using namespace std;

int main()
{
	int num = 10;

	const std::type_info& info1 = typeid(num);

	if (*info1.name() == *"int")
	{
		cout << "num is int type" << endl;
	}

	if (typeid(num) == typeid(int))
	{
		cout << "num is int type" << endl;
	}
	
	return 0;
}
```

결과 :  
<div>
    <img src="/assets/images/rtti/result2.png" alt="matrix" width="70%" min-width="300px" itemprop="image">
</div>