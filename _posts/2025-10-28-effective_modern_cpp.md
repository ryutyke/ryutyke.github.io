---
title: "Effective Modern C++ 요약"
excerpt: "1회 완독 정리 후 반복 학습"

categories:
  - C/C++
  - pfselect
tags:
  - [C, C++]

permalink: /c-cpp/effective_modern_cpp/

toc: true
toc_sticky: true

date: 2025-10-28 20:00:00
last_modified_at: 2025-10-28
---
<br>

- (책에 대한) 설명
    - 좁은 줄임표 “…”는 다른 코드가 들어갈 수 있음을 뜻하며, 넓은 줄임표 “. . .”는 C++ 문법인 줄임표이다.
    - 이동 생성자를 통한 복사본과 복사 생성자를 통한 복사본을 구분하지 않고 복사본이라고 통칭한다.
    - 호출 지점에서 함수에 전달한 표현식을 인수라고 부르고, 인수는 함수의 매개변수를 초기화하는 데 쓰인다. `void someFunc(Widget w);`에서 w는 매개변수이고, `someFunc(wid);` 에서 wid는 인수이다.
    - 일반적으로 함수 객체는 operator() 멤버 함수를 지원하는 형식의 객체를 뜻하지만, 비멤버 함수, 함수 포인터 등까지 포함하는 넓은 의미로 쓰일 수 있다. 그냥 일정한 함수 호출 구문을 이용해서 실행할 수 있는 모든 것이라고 생각해도 무방하다.
    - 람다 표현식을 통해 만들어진 함수 객체를 클로저라고 부른다. 이 책에서는 람다 표현식과 그로부터 생성된 클로저를 통틀어서 람다라고 칭할 수 있다.
    - 함수 템플릿(함수를 산출하는 템플릿)과 템플릿 함수(함수 템플릿으로부터 산출된 함수)를 구분하지 않을 수 있다. 클래스 템플릿도 마찬가지다.
    - 이 책에서는 선언과 정의에서 정의는 선언의 요건들도 갖추고 있으므로 정의라는 점이 중요한 경우가 아닌 한 그냥 선언이라는 용어를 사용한다.
    - 이 책은 함수의 서명(signature)이 함수의 선언 중 매개변수 형식들과 반환 형식을 지정한 부분이라고 정의한다. 함수 이름과 매개변수 이름은 서명에 포함되지 않는다. 예를 들면,`bool(const Widget&)` 이다. noexcept와 constexpr 등도 서명에 포함되지 않는다.
    - C++ 표준에서 deprecate된 기능들은 이후의 표준들에서 언제라도 제거될 수 있다. 예를 들어 C++11에서 std::auto_ptr은 deprecate된 기능이다.
    - 미정의 행동은 연산의 실행시점(runtime)의 행동을 예측할 수 없다는 뜻이다. 벡터의 범위를 벗어나는 대괄호 참조나 data race 등이 있다.
    - 단어 설명
        - 이동 의미론 : move semantics
        - 생 포인터, 똑똑한 포인터 : raw pointer, smart pointer
        - dtor : destructor(소멸자) 줄인 것.
    - 이미 알고 있는 책의 문제점들은 https://www.aristeia.com/BookErrata/emc++-errata.html 에 있다.

- 항목 1 : 템플릿 형식 연역 규칙을 숙지하라
    - auto는 템플릿에 대한 형식 영역을 기반으로 작동한다. 그러므로 auto를 잘 활용하기 위해서는 템플릿 형식 연역(추론)을 이해하는 것이 좋다.
    - 아래 코드에서 컴파일러는 expr을 이용해서 두 가지 형식을 연역한다. T와 ParamType이다. ParamType에는 const나 참조 한정사(&, &&)가 붙을 수 있기 때문에 이 둘의 형식이 서로 다를 수 있다. T의 형식은 expr의 형식과 ParamType의 형태에 의존한다.
        
        ```cpp
        template<typename T>
        void f(ParamType param);
        
        f(expr);
        ```
        
    - ParamType의 형태에 따라 총 세 가지로 나뉜다.
        - ParamType이 참조 형식이지만 보편 참조는 아닌 경우
        - ParamType이 보편 참조인 경우
        - ParamType이 참조가 아닌 경우, 다른 말로 값 전달 (포인터 포함)
    - [경우 1] : ParamType이 참조 형식이지만 보편 참조는 아닌 경우
        - expr이 참조 형식이면 참조 부분을 무시한다.
        - expr의 형식을 ParamType에 대해 패턴 부합 방식으로 대응시켜서 T의 형식을 결정한다.
        
        ```cpp
        template<typename T>
        void f(T& param);
        
        int x = 27;
        const int cx = x;
        const int& rx = x;
        
        f(x);       // T는 int, param의 형식은 int&
        
        f(cx);      // T는 const int,
        				  	// param의 형식은 const int&
        							
        f(rx);      // T는 const int,
        		  			// param의 형식은 const int&
        ```
        
        - const 객체를 참조 매개변수에 전달하는 호출자는 그 객체가 수정되지 않을 것이라고 기대한다. 따라서, 객체의 const성은 T에 대해 연역된 형식에 반영되게 설계됐다.
        - rx의 참조성은 무시됐다.
        - 근데 매개변수 형식을 T&에서 const T&로 바꾸면 const성이 보장되므로 const성이 연역된 형식에 반영되지 않는다.
            
            ```cpp
            template<typename T>
            void f(const T& param);
            
            int x = 27;
            const int cx = x;
            const int& rx = x;
            
            f(x);       // T는 int, param의 형식은 const int&
            
            f(cx);      // T는 int, param의 형식은 const int&
            							
            f(rx);      // T는 int, param의 형식은 const int&
            ```
            
    - [경우 2] : ParamType이 보편 참조인 경우
        - 보편 참조에 대한 내용은 항목 24에 나온다. 보편 참조는 선언 형식은 T&&이며, 좌측값, 우측값을 둘 다 받을 수 있으며 각각에 따라 다르게 행동하는 것이다.
        - 만일 expr이 좌측값이면 T와 ParamType 둘 다 좌측값 참조로 연역된다. 템플릿 형식 연역에서 T가 참조로 연역되는 경우는 이것이 유일하다. 또한, ParamType의 선언 구문은 우측값 참조와 같은 모습이지만 연역된 형식은 좌측값 참조이다.
        - 만일 expr이 우측값이면 경우 1의 규칙들이 적용된다.
        
        ```cpp
        template<typename T>
        void f(const T&& param);
        
        int x = 27;
        const int cx = x;
        const int& rx = x;
        
        f(x);       // x는 좌측값, 따라서 T는 int&
        						//param의 형식 역시 int&
        
        f(cx);      // cx는 좌측값, 따라서 T는 const int&
        						// param의 형식 역시 const int&
        							
        f(rx);      // rx는 좌측값, 따라서 T는 const int&
        						// param의 형식 역시 const int&
        
        f(27);      // 27은 우측값, 따라서 T는 int
        						// param의 형식은 int&&
        ```
        
    - [경우 3] : ParamType이 참조가 아닌 경우, 다른 말로 값 전달 (포인터 포함)
        - 값 전달의 경우 param은 주어진 인수의 복사본, 즉 새로운 객체이다. 따라서 T가 연역될 때 참조와 const성이 무시된다. (volatile도 무시된다.)
            
            ```cpp
            template<typename T>
            void f(T param);
            
            int x = 27;
            const int cx = x;
            const int& rx = x;
            
            f(x);       // T는 int, param의 형식은 int
            
            f(cx);      // T는 int, param의 형식은 int
            							
            f(rx);      // T는 int, param의 형식은 int
            ```
            
        - 포인터에 대해서도 똑같은데, const에 대해 한 가지 주의할 점이 있다. 포인터 오른쪽에 있는 const는 포인터 자체에 대한 const이고 왼쪽에 있는 const는 포인터가 가리키는 것이 const라는 뜻인데, 이때 포인터 자체에 대한 우측 const는 무시되며 포인터가 가리키는 것에 대한 좌측 const는 적용된다.
            
            ```cpp
            template<typename T>
            void f(T param);
            
            const char* const ptr = "hi";
            
            f(ptr);      // T, param의 형식은 const char*
            ```
            
    - 배열에 대한 이야기다. 배열은 배열의 첫 원소를 가리키는 포인터로 붕괴된다.
        
        ```cpp
        const char name[] = "hi"; // name의 형식은 const char[3]
        
        const char* ptrToName = name; // 배열이 포인터로 붕괴된다.
        ```
        
    - 배열을 값 전달 매개변수를 받는 템플릿에 전달하면 포인터로 붕괴된다.
        
        ```cpp
        void myFunc(int param[]); // 적법한 구문. 아래와 같이 취급되ㅣㄴ다.
        void myFunc(int* param);
        
        myFunc(name); // T는 const char*
        ```
        
    - 이처럼 함수의 매개변수를 진짜 배열로 취급하게 할 수는 없지만, 배열에 대한 참조로 선언할 수는 있다. 템플릿 인수를 참조로 받도록 하고 배열을 전달하면 된다. 이를 활용하면 배열에 담긴 원소들의 개수를 연역하는 템플릿을 만들 수도 있다. (constexpr을 통해 컴파일 도중 사용할 수 있게 한다. 항목 15 참고)
        
        ```cpp
        template<typename T>
        void f(T& param);
        
        f(name); // T는 const char[3], param의 형식은 const char (&)[3]
        
        ====================
        // 배열의 크기를 컴파일 시점 상수로서 돌려주는 템플릿 함수
        template<typename T, std::size_t N>
        constexpr std::size_t arraySize(T (&)[N]) noexcept
        {
        	return N;
        }
        
        int keyVals[] = { 1, 3, 7, 9, 11 };
        int mappedVals[arraySize(keyVals)];
        std::array<int, arraySize(keyVals)> mappedVals;
        ```
        
    - 배열뿐만 아니라 함수 형식도 함수 포인터로 붕괴된다.
        
        ```cpp
        void someFunc(int, double);
        
        template<typename T>
        void f1(T param);
        
        template<typename T>
        void f2(T& param);
        
        f1(someFunc);    // param의 형식은 void (*)(int, double). 함수 포인터
        
        f2(someFunc);    // param의 형식은 void (&)(int, double). 함수 참조
        ```