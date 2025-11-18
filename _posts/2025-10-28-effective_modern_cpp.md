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
        - 중복적재 : 오버로딩
        - 연결 목록 : linked list
        - 배정 : 대입
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
        
        f(x);     // T는 int, param의 형식은 int&
        
        f(cx);    // T는 const int,
                  // param의 형식은 const int&
        							
        f(rx);    // T는 const int,
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
            
            f(x);     // T는 int, param의 형식은 const int&
            
            f(cx);    // T는 int, param의 형식은 const int&
            							
            f(rx);    // T는 int, param의 형식은 const int&
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
        
        f(x);     // x는 좌측값, 따라서 T는 int&
                  // param의 형식 역시 int&
        
        f(cx);    // cx는 좌측값, 따라서 T는 const int&
                  // param의 형식 역시 const int&
        							
        f(rx);    // rx는 좌측값, 따라서 T는 const int&
                  // param의 형식 역시 const int&
        
        f(27);    // 27은 우측값, 따라서 T는 int
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
            
            f(x);     // T는 int, param의 형식은 int
            
            f(cx);    // T는 int, param의 형식은 int
            							
            f(rx);    // T는 int, param의 형식은 int
            ```
            
        - 포인터에 대해서도 똑같은데, const에 대해 한 가지 주의할 점이 있다. 포인터 오른쪽에 있는 const는 포인터 자체에 대한 const이고 왼쪽에 있는 const는 포인터가 가리키는 것이 const라는 뜻인데, 이때 포인터 자체에 대한 우측 const는 무시되며 포인터가 가리키는 것에 대한 좌측 const는 적용된다.
            
            ```cpp
            template<typename T>
            void f(T param);
            
            const char* const ptr = "hi";
            
            f(ptr);     // T, param의 형식은 const char*
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
        
        f1(someFunc);   // param의 형식은 void (*)(int, double). 함수 포인터
        
        f2(someFunc);   // param의 형식은 void (&)(int, double). 함수 참조
        ```

- 항목 2 : auto의 형식 연역 규칙을 숙지
    - auto를 이용해서 변수를 선언할 때는 auto는 템플릿의 T와 동일한 역할을 하며, 변수의 형식 지정자는 ParamType과 동일한 역할을 한다.
        
        ```cpp
        auto x = 27; // 형식 지정자는 auto
        
        const auto cx = x; // 형식 지정자는 const auto
        
        const auto& rx = x; // 형식 지정자는 const auto&
        ```
        
    - 위 예제에 대해, 컴파일러는 auto 선언마다 템플릿 함수 하나와 해당 초기화 표현식으로 그 템플릿 함수를 호출하는 구문이 존재하는 것처럼 행동한다.
        
        ```cpp
        template<typename T>
        void func_for_x(T param);
        
        func_for_x(27);
        
        template<typename T>
        void func_for_cx(const T param);
        
        func_for_cx(x);
        
        template<typename T>
        void func_for_rx(const T& param);
        
        func_for_rx(x);
        ```
        
    - 딱 한 가지만 빼고, auto에 대한 형식 연역은 위 규칙에 따라 템플릿 형식 연역과 동일하게 동작한다. 다른 한 가지는 균일 초기화이다. auto로 선언된 변수의 초기치(initializer)가 중괄호 형태면 std::initializer_list로 연역된다. 템플릿 매개변수 T는 std::initializer_list로 연역하지 못 한다.
        
        ```cpp
        auto x1 = 27;     // 형식은 int, 값은 27 
        auto x2(27);      // 형식은 int, 값은 27
        auto x3 = { 27 }; // 형식은 std::initializer_list<int> 값은 {27}
        auto x4{ 27 };    // 형식은 std::initializer_list<int> 값은 {27}
        
        auto x5 = { 1, 2, 3.0 }; // 컴파일 오류. initializer_list는 암시적 변환 안 됨.
        
        ====
        
        template<typename T>
        void f(T param);
        
        f({ 11, 23, 9 }); // 컴파일 오류. T는 std::initializer_list로 연역되지 않음.
        
        template<typename T>
        void f(std::initializer_list<T> param);
        
        f({ 11, 23, 9 }); // 연역 성공. T는 int.
        ```
        
    - (2014년에 “=”가 없는 형태인 직접 초기화 구문을 이용한 auto 중괄호 초기치에 대한 해당 형식 연역 규칙을 제거하자는 제안을 C++ 표준이 받아들였다고 한다. 따라서 이를 적용한 컴파일러에서는 x4에서의 auto는 int이다.) 아래 코드를 실행해 보니, 내 컴파일러에는 적용되어 있었다.
        
        ```cpp
        #include<bits/stdc++.h>
        
        int main() 
        {
        	auto x4{ 27 }; // int
        	// auto x4{ 27, 2 }; // 컴파일 오류
        	// auto x4 = { 27, 2 }; // class std::initializer_list<int>
        	
        	std::cout << typeid(x4).name();
        }
        ```
        
    - C++14에서는 함수의 반환 형식을 auto로 지정해서 컴파일러가 연역하게 만들 수 있으며, 람다의 매개변수 선언에 auto를 사용하는 것도 가능하다. 그러나 auto의 그러한 용법들에는 auto 형식 연역이 아니라 템플릿 형식 연역의 규칙들이 적용된다.
        
        ```cpp
        auto createInitList()
        {
        	return { 1, 2, 3 }; // 컴파일 오류.
        }
        
        std::vector<int> v;
        
        auto resetV =
        	[&v](const auto& newValue) { v = newValue; };
        
        resetV({ 1, 2, 3 }); // 컴파일 오류.
        ```

- 항목 3 : decltype의 작동 방식을 숙지하라
    - decltype(**decl**ared **type**)은 주어진 이름이나 표현식의 구체적인 형식을 알려준다.
    - decltype은 함수의 반환 형식이 그 매개변수 형식들에 의존하는 함수 템플릿을 선언할 때 주로 쓰인다. 아래 예제에서 반환 형식에 있는 auto는 형식 연역과는 아무런 관련이 없다. C++11의 후행 반환 형식을 쓰겠다는 의미일 뿐이다. 후행 반환 형식 구문은 반환 형식을 매개변수들을 이용해서 지정할 수 있다는 장점이 있다.
        
        ```cpp
        template<typename Container, typename Index>
        auto authAndAccess(Container& c, Index i)
        	-> decltype(c[i])
        {
        	authenticateUser();
        	return c[i];
        }
        ```
        
    - auto를 사용한 반환 형식의 연역도 있긴 하다. C++11은 람다 함수가 한 문장으로 이루어져 있다면 그 반환 형식의 연역을 허용하며,  C++14는 모든 람다와 모든 함수의 반환 형식 연역을 허용한다. return문이 여러 개면 모든 return문의 형식 연역 결과가 일치해야 한다.
        
        ```cpp
        template<typename Container, typename Index>
        auto authAndAccess(Container& c, Index i)
        {
        	authenticateUser();
        	return c[i];        // c[i]로부터 반환 형식 연역
        }
        ```
        
    - 근데 이렇게 했을 경우, auto는 템플릿 형식 연역과 동일하게 작동하는데, 컨테이너의 operator[ ] 연산의 반환 형식인 T&에서 참조성이 무시된다는 문제가 있다.
        
        ```cpp
        authAndAccess(d, 5) = 10; // 참조성 무시로 인해 우측값에 우측값을 넣게 되어 오류
        ```
        
    - **C++14에는** decltype(auto) 지정자가 있다. 여기서 auto는 형식이 연역되어야 함을 뜻하고 decltype은 그 연역 과정이 decltype 형식 연역 규칙으로 진행되어야 함을 뜻한다.
        
        ```cpp
        template<typename Container, typename Index>
        decltype(auto) authAndAccess(Container& c, Index i)
        {
        	authenticateUser();
        	return c[i];        // c[i]로부터 반환 형식 연역
        }
        ```
        
    - decltype(auto) 지정자는 함수 반환 형식에만 사용할 수 있는 것은 아니다.
        
        ```cpp
        Widget w;
        const Widget& cw = w;
        auto myWidget1 = cw;   // 형식 : Widget (참조성 무시)
        decltype(auto) myWidget2 = cw;   // 형식 : Widget&
        ```
        
    - 현재 항목의 주제와 다른 내용이지만, 위의 예제를 발전시킬 수 있다. 컨테이너 매개변수를 현재 좌측값 참조로 두었기에 우측값으로 줄 수 없다. 보편 참조를 쓰면 좌측값 우측값 모두 쓸 수 있는 매개변수를 사용할 수 있다. 그리고 반환할 때는 인자로 들어온 표준 라이브러리가 사용하는 방식을 따르도록 std::forward를 쓸 수 있다. std::forward는 인자의 값 전달 방식을 따라가는 문법이다. 좌측값이면 좌측값, 우측값이면 우측값. std::move는 무조건 우측값.
        
        ```cpp
        template<typename Container, typename Index>
        decltype(auto) authAndAccess(Container&& c, Index i) // 보편참조
        {
        	authenticateUser();
        	return std::forward<Container>(c)[i]; // std::forward 사용
        }
        ```
        
    - decltype이 아주 가끔 뜻밖의 형식을 연역하기도 한다. decltype을 이름에 적용하면 그 이름에 대해 선언된 형식이 산출된다. 그런데 이름보다 복잡한 왼값 표현식에 대해서는 일반적으로 왼값 참조로 산출된다. 간단한 예를 들면, int x에 대해 decltype(x)는 int인데, decltype((x))는 int&이다.

- 항목 4 : 연역된 형식을 파악하는 방법을 알아두라
    - IDE 코드 편집기 중에 마우스 커서를 올리면 그 개체의 형식을 표시해 주는 것이 있다. 이는 IDE 안에서 C++ 컴파일러가, 적어도 앞단이 실행되기 때문이다.
    - 형식 때문에 컴파일에 문제가 발생하게 만드는 방법도 있다. 보통 오류 메시지에는 문제를 일으킨 형식이 나온다.
        
        ```cpp
        template<typename T> // 정의 없이 선언만 해둔다.
        class TD;
        
        TD<decltype(x)> xType; // x 형식이 담긴 오류 메시지가 나온다. 
        ```
        
    - 런타임에 typeid로 std::type_info 객체를 받아, std::type_info::name을 사용하여 로깅하는 방법도 있다. 그러나 std::type_info::name은 주어진 형식을 마치 템플릿 함수에 값 전달 매개 변수로서 전달된 것처럼 취급해야 해서 참조성 무시, const성 무시가 발생한다. boost::typeindex::type_id_with_cvr.pretty_name()은 그렇지 않다.
        
        ```cpp
        std::cout << typeid(x).name();
        
        =======
        
        #include <boost/type_index.hpp>
        
        template<typename T>
        void f(const T& param)
        {
        	using boost::typeindex::type_id_with_cvr;
        	
        	std::cout << type_id_with_cvr<T>().pretty_name();
        	std::cout << type_id_with_cvr<decltype(param)>().pretty_name();
        
        }
        ```
        
    - 정확하지 않을 수도 있으므로, 형식 연역 규칙을 제대로 이해하자!

- 항목 5 : 명시적 형식 선언보다는 auto를 선호하라
    - auto를 쓰면 변수의 초기화를 빼먹는 실수가 사라진다.
    - 클로저를 담는 변수로 auto 대신 std::function을 쓰면 되지 않나? ⇒ 일반적으로 std::function이 auto보다 메모리와 시간을 더 많이 소비하며, 때에 따라서는 메모리 부족 예외를 유발할 수도 있다.
    - auto를 쓰면 형식 단축(type shortcut) 문제를 피할 수 있다. v.size()의 반환 형식은 std::vector<int>::size_type인데 unsigned로 받았다. 64비트에서 std::vector<int>::size_type은 64비트이지만 unsigned는 32비트이다.
        
        ```cpp
        std::vector<int> v;
        unsigned sz = v.size();
        ```
        
    - 아래 예시와 같은 실수도 피할 수 있다. 해시맵의 key는 const이다. 아래 예시처럼 const를 빼먹는 실수를 할 수 있다.
        
        ```cpp
        std::unordered_map<std::string, int> m;
        
        for (const std::pair<std::string, int>& p : m)
        {
        	// ...
        }
        ```
        
    - auto를 쓰면 입력도 편하고, 리팩토링도 수월해질 수 있다. 형식을 바꿀 때 auto면 많이 안 바꿔도 된다.
    - 그래도 auto를 사용하면 가독성 문제가 있을 수 있고, 항목 2, 6 내용도 고려해야 한다는 단점도 있다.

- 항목 6 : auto가 원치 않은 형식으로 연역될 때에는 명시적 형식의 초기치를 사용하라
    - std::vector<bool>의 operator[ ]가 돌려주는 것은 그 컨테이너의 한 요소에 대한 참조가 아니라 std::vector<bool>::reference 형식의 객체이다. (std::vector<bool> 안에 내포된 대리자 클래스) std::vector<bool>이 자신의 bool들을 1비트로 표현하도록 하는데, std::vector<T>의 operator[ ] 반환 형식이 T&지만, C++에서 비트에 대한 참조는 금지되어 있다. 따라서 마치 bool&처럼 작동하는 객체를 돌려주는 우회책을 사용하는 것이다.
    
    ```cpp
    std::vector<bool> features(const Widget& w);
    
    Widget w;
    bool highPriority = features(w)[5]; // 암시적 형변환
    processWidget(w, highPriority);
    
    auto highPriority = features(w)[5]; // 임시 객체인 벡터의 대리자 클래스의 비트 포인터
    processWidget(w, highPriority);     // 임시 객체 벡터 사라지면서 미정의 행동
    ```
    
    - auto가 위의 예시처럼 대리자 클래스의 형식을 연역할 때는 auto가 다른 형식을 연역하도록 강제하는 방법을 쓸 수 있다. 형식을 명시적으로 지정한 초기치 관용구, 형식 명시 초기치 관용구를 사용하면 된다. auto로 선언하되, 초기화 표현식의 형식을 auto가 연역하길 원하는 형식으로 캐스팅해 주는 것이다.
        
        ```cpp
        auto highPriority = static_cast<bool>(features(w)[5]);
        ```
        
    - 형식 명시 초기치 관용구를 위의 예시 같은 상황에서만 쓸 수 있는 것이 아니다. 예를 들어 float의 정밀도로도 충분해서 double을 float로 넣을 때 명시하여 의도를 명확히 하는 것이 있다.
        
        ```cpp
        double calcEpsilon();
        // float ep = calcEpsilon(); // 암시적
        auto ep = static_cast<float>(calcEpsilon()); // 명시하여 의도 명확화
        ```

- 항목 7 : 객체 생성 시 괄호와 중괄호를 구분하라
    - = { } 와 같이 등호와 중괄호를 같이 사용한 구분은 중괄호만 사용한 구문과 동일하게 취급한다.
    - 중괄호 초기화는 C++11에서 균일 초기화라는 이름으로 도입됐다.
    - 중괄호 초기화의 장점은 C++의 세 가지 초기화 표현식 방법 중 유일하게 어디서나 범용적으로 사용할 수 있다는 것, 암묵적 좁히기 변환을 방지해 준다는 점, C++의 가장 성가신 구문 해석에 자유롭다는 점이다.
        - 좁히기 변환 : double을 int에 넣을 때 괄호나 등호를 사용한 초기화는 이를 허용함. 중괄호 초기화는 허용하지 않음.
        - 가장 성가신 구문 해석 : `Widget w2();` 이 인수 없는 생성자를 호출하는 것이 아니라 함수 선언으로 취급된다. `Widget w3{};` 는 인수 없는 생성자로 취급된다.
    - 단점도 있다. 특히, `std::initializer_list`를 받는 생성자 오버로딩이 있을 경우, 웬만해서 컴파일러는 그것을 사용하려 한다.
        
        ```cpp
        class Widget {
        public:
        	Widget(int i, double d);
        	Widget(std::initializer_list<bool> il);
        };
        
        Widget w{10, 5.0}; // 오류. initializer_list 생성자 사용. 하지만 좁히기 변환 안 됨.
        ```
        
    - 생성자를 설계할 때는 괄호를 사용하느냐 중괄호를 사용하느냐에 따라 서로 다른 오버로딩 버전이 선택되는 일이 없도록 하는 것이 최선이다. std::vector는 그렇게 설계하지 못 한 예시이다. std::vector는 `v1(10, 20)`과 `v2{10, 20}`의 의미가 완전히 다르다. 20의 값이 10개인 벡터 생성과 10, 20을 원소로 가진 벡터 생성. 이에 연결되는 문제는 아래 예시이다.
        
        ```cpp
        // 임의의 개수의 인수들을 지정해서 임의의 형식의 객체를 생성
        template<typename T, typename... Ts>
        void doSomeWork(Ts&&... params)
        {
        	// T localObject(std::forward<Ts>(params)...);  괄호 사용
        	// T localObject{std::forward<Ts>(params)...};  중괄호 사용
        }
        
        doSomeWork<std::vector<int>>(10, 20); // doSomeWork 구현에 따라 아예 다른 결과
        ```
        
    - 위의 예시에서 템플릿으로부터 만들어진 함수가 괄호를 사용할 것인지 아니면 중괄호를 사용할 것인지를 호출자가 결정할 수 있는 유연한 설계도 가능하다고 한다. 요약하면 tag를 사용하는 것이다. 이미 다른 std에서는 tag를 사용하고 있다.사이트 참고 : https://akrzemi1.wordpress.com/2013/06/05/intuitive-interface-part-i/
        
        ```cpp
        namespace std{
          constexpr struct with_size_t{} with_size{};
          constexpr struct with_value_t{} with_value{};
          constexpr struct with_capacity_t{} with_capacity{};
        }
        
        	
        std::vector<int> v1(std::with_size, 10, std::with_value, 6);
        std::vector<int> v2{std::with_size, 10, std::with_value, 6};
        ```
        

- 항목 8 : 0과 NULL보다 nullptr를 선호하라
    - 0은 int이지 포인터가 아니다.
    - NULL은 컴파일러에 따라 int, long 등 정수 형식으로 처리되고, 마찬가지로 포인터 형식이 아니다.
    - nullptr은 모든 형식의 로우 포인터 형식으로 암묵적 변환이 될 수 있는 std::nullptr_t이다. 결코 정수 형식으로는 해석되지 않는다.
    - 코드 설계 시, nullptr을 안 쓰고 0 또는 NULL을 사용하는 사람이 있을 수 있으므로, 정수 형식과 포인터 형식에 대한 중복적재를 피하라.

- 항목 9 : typedef보다 별칭 선언(using)을 선호하라
    - 별칭 선언은 using 구문을 뜻한다. typedef와 using이 하는 일은 정확히 동일하다. 그러나 책에서는 using이 더 좋은 두 가지 이유를 알려준다.
    - 첫 번째는, 함수 포인터를 다룰 때 직관적이다.
        
        ```cpp
        typedef void (*FP)(int, const std::string&);
        using FP = void (*)(int, const std::string&);
        ```
        
    - 두 번째는, typedef는 템플릿화할 수 없지만, using은 가능하다. 템플릿화된 별칭 선언을 별칭 템플릿(alias templates)이라고 부른다. typedef는 편법을 써야 가능하다.
        
        ```cpp
        template<typename T>
        using MyAllocList = std::list<T, MyAlloc<T>>;
        
        MyAllocList<Widget> lw;
        
        =================
        // 템플릿화된 strcut 안에 typedef를 사용하는 편법을 써야 한다.
        template<typename T>
        struct MyAllocList {
        	typedef std::list<T, MyAlloc<T>> type;
        };
        ```
        
    - 또한 중첩 의존 이름(이펙티브 C++ 항목 42 참고)에 대해 typedef는 typename 처리를 해야 하고, using은 안 한다.
        
        ```cpp
        template<typename T>
        class Widget {
        private:
        	MyAllocList<T> list;
        };
        
        =================
        // typename, ::type을 써야 한다.
        template<typename T>
        class Widget {
        private:
        	typename MyAllocList<T>::type list;
        };
        ```
        
    - 표준 위원회도 C++11의 모든 형식 변화에 대한 별칭 템플릿 버전들을 C++14에 포함시켰다. 예시에 있는 형식 변화 문법들은 템플릿 메타프로그래밍에서 흔히 쓰인다.
        
        ```cpp
        ste::remove_const<T>::type
        std::remove_const_t<T> // C++ 14
        
        ste::remove_reference<T>::type
        std::remove_reference_t<T> // C++ 14
        
        std::add_lvalue_reference<T>::type
        std::add_lvalue_reference_t<T> // C++ 14
        ```

- 항목 10 : 범위 없는 enum보다 위 있는 enum을 선호하라
    - 범위 없는(unscoped) enum은 그냥 enum이고, 범위 있는(scoped) enum은 enum class를 말한다.
    - 범위 없는 enum은 열거자 이름들이 자신을 정의하는 enum의 범위로 새어 나간다는 단점이 있다. namespace 오염이 생긴다.
        
        ```cpp
        enum Color { black, white, red };
        
        auto white = false; // 오류. 이미 white가 선언되어 있음.
        
        =======
        
        enum class Color { black, white, red };
        
        auto white = false; // Ok.
        ```
        
    - 범위 없는 enum은 암묵적 형식 변환이 일어나지만, **범위 있는 enum은 그렇지 않다**. 다른 형식으로 변환하고 싶으면 캐스팅을 하면 된다.
        
        ```cpp
        enum class Color { black, white, red };
        
        Color c = white; // 오류. 암묵적 형식 변환 안 된다.
        Color c = Color::white; // Ok.
        
        if (static_cast<double>(c) < 14.5) // 캐스팅
        ```
        
    - 범위 있는 enum은 전방 선언이 가능하다. (범위 없는 enum도 가능하긴 하다.) 전방 선언을 할 수 있으려면 컴파일 타임에 쓰이기 전에도 크기를 알 수 있어야 한다. enum class는 기본 바탕 형식이 int로 정해져 있기 때문에 전방 선언이 가능하다. 범위 없는 enum은 기본 바탕 형식이 없다. 둘 다 바탕 형식을 명시적으로 지정이 가능하다. 명시적으로 지정한 경우엔 범위 없는 enum도 전방 선언이 가능하다.
        
        ```cpp
        enum class Status : std::uint32_t;
        
        enum Color : std::uint8_t;
        ```
        
    - 범위 없는 enum이 유용한 경우는 tuple과 함께 사용할 때이다. 암묵적 변환이 되므로 편한 경우이다.
        
        ```cpp
        enum UserInfoFields {uiName, uiEmail, uiReputation };
        UserInfo uInfo;
        
        auto val = std::get<uiEmail>(uInfo);
        ```
        
    - static_cast를 쓰면 enum class도 이렇게 사용이 가능하다. 근데 캐스팅만 하면 되는 게 아니라, 바탕 형식을 맞춰서 돌려줘야 한다. 이때 std::underlying_type 형식 특질을 사용하면 된다. 이는 enum이나 enum class가 내부에서 어떤 정수 타입을 기반으로 저장되는지 추출하는 C++ 표준 타입 trait이다.
        
        ```cpp
        // C++ 14 기준. auto, type_t 사용
        template<typename E>
        constexpr auto toUType(E enumerator) noexcept
        {
        	return static_cast<std::underlying_type_t<E>>(enumerator);
        }
        ```
        
    - 좀 더 코드가 길어지지만, 그래도 enum class를 쓰는 편이 좋은 것 같다.

- 항목 11 : 정의되지 않은 비공개 함수보다 삭제된 함수를 선호하라
    - 특정 함수를 호출하지 못하게 하는 가장 쉬운 방법은 그 함수를 선언하지 않는 것이다. 그러나 C++(컴파일러)이 자동으로 선언하는 경우가 있다. 그 중, 흔히 사용을 금지하려는 멤버 함수는 복사 생성자, 복사 대입 연산자이다.
    - C++98에서는 private로 선언하고 정의를 하지 않는 방식을 사용했다. private에 선언되어 있어서 호출할 수 없고, 만약 접근 가능한 곳에서 이를 호출한다고 해도 정의가 없어서 링크가 실패한다.
    - C++11에서는 “= delete”를 사용할 수 있다. 이때 함수를 public에 선언하는 것이 관례이다. 이는 C++이 함수의 접근성을 점검한 후에야 삭제(= delete) 여부를 확인하기 때문이다. private에 두면 함수 사용이 문제될 때, 삭제된 것이 이유가 아니라 private임이 이유가 될 수 있기 때문이다. 이는 오해의 여지를 제공한다.
    - 함수 삭제를 이용하면 private과 달리 링크 시점이 아니라 컴파일 타임에 오류를 확인할 수 있다.
    - private 방식은 멤버 함수에만 적용할 수 있으나, 함수 삭제는 그 어떤 함수에도 적용이 가능하다. 함수 템플릿 인스턴스 삭제도 가능하다.
    - 아래 예제에서 double 오버로드를 삭제하면 float까지 삭제된다. 정확히는 float 값을 인자로 넣으면 이것이 int가 아니라 double로 암시적 형변환이 이뤄지고, double 버전이 삭제되었기 때문에 컴파일에 실패한다.
        
        ```cpp
        bool Func(int number);
        bool Func(char) = delete;
        bool Func(bool) = delete;
        bool Func(double) = delete;
        ```
        

- 항목 12 : 재정의 함수들을 override로 선언하라
    - 재정의가 일어나는 필수조건들이다.
        - 기반 클래스 함수가 가상 함수이어야 한다
        - 기반 함수와 이름이 동일해야 한다
        - 매개변수 형식들이 동일해야 한다
        - const성이 동일해야 한다
        - 반환 형식과 예외 명세가 호환되어야 한다
        - **(C++11 추가)** 참조 한정사가 동일해야 한다.
    - 이와 같이 재정의 조건들이 많기 때문에, 실수할 수 있다. 재정의 되었다고 생각하지만 되지 않는 경우를 말하는 것이다.
    - override로 선언하면, 재정의를 의도한 함수가 실제로는 아무것도 재정의하지 않을 때 컴파일 오류가 발생한다.
    - 함수의 참조 한정사는 아래 예제처럼 멤버 함수가 호출되는 객체를 한정하는 것이다.
        
        ```cpp
        class Widget {
        public:
        	void doWork() &;  // *this가 좌측값일 때만 사용
        	void doWork() &&; // *this가 우측값일 때만 사용
        };
        ```
        
    - 임시 객체(우측값)에 대해 호출됐을 경우 복사가 아니라 이동을 하는 아래 같은 예제에서 활용할 수 있다.
        
        ```cpp
        class Widget {
        public:
        	using DataType = std::vector<double>;
        	
        	DataType& data() & { return values; }
        	DataType& data() && { return std::move(values); }
        	
        private:
        	DataType values;
        };
        
        auto vals1 = w.data(); // 복사 생성
        auto vals2 = makeWidget().data(); // 이동 생성
        ```

- 항목 13 : iterator보다 const_iterator를 선호하라
    - 가능한 한 const를 사용하라는 것은 iterator에도 적용된다.
    - C++98에서는 비const 컨테이너로부터 const_iterator를 얻는 간단한 방법이 없었다. 또한, 삽입/삭제 위치를 iterator로만 지정할 수 있었다. const_iterator는 허용되지 않았다. 근데 const_iterator는 iterator로 변환되지 않는다.
    - C++11에서는 const_iterator를 얻기도 쉽고 사용하기도 쉽다. 컨테이너 멤버 함수 cbegin과 cend는 const_iterator를 반환한다. 그리고 삽입/삭제 위치를 지정하는 목적으로 iterator를 사용하는 STL 멤버 함수(insert, erase)는 const_iterator를 사용한다.
        
        ```cpp
        std::vector<int> values;
        auto it = std::find(values.cbegin(), values.cend(), 1983);
        values.insert(it, 1998);
        ```
        
    - C++11에서는 비멤버 함수 begin과 end는 표준에 추가했지만, cbegin, cend, rbegin 등은 그렇지 않다. C++14에는 있다. 따라서 C++11 일 경우에는, 제너릭 라이브러리 코드를 작성할 때 cbegin이나 cend를 비멤버 함수로서 제공해야 하는 컨테이너가 있음을 고려해서 작성해야 한다. 아래 예제처럼  C++11에서도 제공하는 비멤버 begin을 사용해서 비멤버 cbegin 함수를 만들 수 있다.
        
        ```cpp
        template <class C>
        auto cbegin(const C& container)->decltype(std::begin(container)) // const 파라미터
        {
        	return std::begin(container); // const 컨테이너의 begin은 const_iterator
        }
        ```
        
    - 위 템플릿은 내장 배열에 대해서도 작동한다. 그런 경우 container 인자는 const 배열에 대한 참조가 될 것이다. 비멤버 begin은 내장 배열에 대해서도 작동하며 주어진 배열의 첫 원소를 가리키는 포인터를 돌려준다. const 배열의 원소들은 const이므로 정상적으로 작동한다.`

- 항목 14 : 예외를 방출하지 않을 함수는 noexcept로 선언하라
    - C++11부터 예외에 대해 의미 있는 정보는 함수가 예외를 하나라도 던질 수 있는지 아니면 절대 던지지 않는지라는 이분법적 정보뿐이라고 판단했다. 그렇게 noexcept 키워드가 생겼다. 그러면서 C++98 스타일의 예외 명세(throw)는 deprecate 기능으로 분류되었다. 또한, 빈 예외 명세 ( throw() )도 deprecate 되었다.
        
        ```cpp
        int f(int x) throw(); // C++98
        int f(int x) noexcept; // C++11
        ```
        
    - 왜일까? 컴파일러 최적화 때문이다. 컴파일러의 최적화기(optimizer)가, C++98 방식은 예외가 발생하면 예외 명세를 위반했는지 검사하기 위해 f를 호출한 지점에 도달할 때까지 **언와인딩을 해야 한다**. 빈 명세인 경우에도 동일하게 진행하도록 설계됐다. 이처럼 C++98 방식은 런타임에 예외에 대해 검사하는 방식이라 동적 예외 명세라고 한다. C++11 방식은 예외 명세를 확인할 필요가 없기 때문에 **언와인딩을 안 해도 된다**. 따라서 언와인딩을 위한 메타데이터를 저장하지 않는 등의 컴파일러 최적화가 가능하다.
    - std::vector에서 push_back 함수에 대해 공간이 부족할 때 더 큰 메모리로 옮기는 과정에서, 기존 방식은 새 메모리로 요소들을 복사하고 기존 메모리에 있는 객체를 파괴하는 방식이라 강한 예외 안전성을 보장했다. 그러나 C++11에서 move가 생기면서 예외 안전성 보장이 위반될 수 있게 됐다. 이처럼 std::vector::push_back이나 표준 라이브러리의 여러 함수는 “가능하면 이동하되 필요하면 복사한다” 전략을 사용한다. 이는 이동 연산이 예외를 방출하지 않음이 확실한 경우에만 복사 연산을 이동 연산으로 대체하는 것이다. 이동 연산이 예외를 방출하지 않음을 어떻게 알아낼 수 있을까? 주어진 이동 연산이 noexcept로 선언되어 있는지를 점검한다.
    - 표준 라이브러리에 있는 swap의 noexcept 여부는 사용자 정의 swap의 noexcept 여부에 어느 정도 의존한다.
        
        ```cpp
        template <class T, size_t N>
        void swap(T (&a)[N], T(&b)[N]) noexcept(noexcept(swap(*a, *b)));
        ```
        
    - noexcept의 정확성이 매우 중요하다. 함수의 구현이 예외를 방출하지 않는다는 성질을 오랫동안 유지할 결심이 선 경우에만 함수를 noexcept로 선언하자. 대부분의 함수가 예외에 중립적이다. 예외 중립적 함수는 스스로 예외를 던지지는 않지만, 예외를 던지는 다른 함수들을 호출할 수는 있는 경우이다. 이는 결코 noexcept가 될 수 없다.
    - 함수를 noexcept로 선언하기 위해 함수의 구현을 작위적으로 변경하는 것의 오버헤드가 noexcept를 통해서 가능한 최적화가 주는 성능 향상을 능가할 수 있음을 인지하자.
    - 기본적으로 모든 메모리 해제 함수와 모든 소멸자는 암묵적으로 noexcept이다. 따라서 그런 함수들은 noexcept로 선언할 필요가 없다. (직접 선언해도 해가 되지는 않는다.)

- 항목 15 : 가능하면 항상 constexpr을 사용하라
    - constexpr 객체는 const이며, 컴파일 도중에 알려지는 값들로 초기화된다. 이렇게 컴파일 시점에서 알려지는 값들은 읽기 전용 메모리에 배치될 수 있다. 또한, 정수 상수 표현식이 요구되는 문맥에서 사용할 수 있다. 예를 들면, 배열 크기, 정수 템플릿 인수(std::array 객체의 길이), 열거자 값 등이 있다.
    - const는 반드시 컴파일 시점에서 알려지는 값으로 초기화되는 것은 아니기 때문에 constexpr과 다르다.
    - constexpr 함수는 컴파일 시점 상수를 인수로 해서 호출된 경우에는 컴파일 시점 상수를 산출한다. 런타임에 알려지는 값으로 호출하면 보통의 함수처럼 런타임에 계산하여 런타임 값을 산출한다. 즉, 함수를 컴파일 시점 상수를 위한 버전과 아닌 경우의 버전으로 나누어서 구현할 필요가 없다. 컴파일 시점 상수를 요구하는 문맥에 constexpr 함수를 사용할 수 있다. 그런 경우에는 인수의 값이 컴파일 시점에서 알려지지 않는다면 컴파일 오류가 발생한다.
    - constexpr 함수는 반드시 리터럴 형식들을 받고 돌려주어야 한다. 리터럴 형식은 컴파일 도중에 값을 결정할 수 있는 형식이다. C++11에서 void를 제외한 모든 내장 형식이 리터럴 형식에 해당한다. 즉, 반환 형식이 void일 수 없다. (C++14에서는 void가 리터럴 형식에 속하게 되었다.)
    - C++11에서는 constexpr 함수에 제약이 있다. 실행 가능 문장이 많아야 하나이어야 한다는 것이다. 반환 형식이 void일 수 없으므로 많아야 하나있는 문장은 보통의 경우 return문일 수밖에 없다. 요령을 이용하면 확장할 수 있는데 하나는 조건부 연산자 ?:를 if-else 문 대신 사용하는 것이고, 또 하나는 루프 대신 재귀를 사용하는 것이다. (또한 C++11에서 constexpr 함수는 return문이 최대 하나여야 한다.)
        
        ```cpp
        // C++11
        constexpr int pow(int base, int exp) noexcept
        {
        	return (exp == 0 ? 1 : base * pow(base, exp - 1));
        }
        ```
        
    - C++14에서는 해당 제약이 사라졌다. void가 리터럴 형식에 속하고, return 문이 최대 한 개도 아니고, 실행 가능 문장이 많아야 하나도 아니다.
        
        ```cpp
        // C++14
        constexpr int pow(int base, int exp) noexcept
        {
        	auto result = 1;
        	for (int i = 0; i < exp; ++i) result *= base;
        	
        	return result;
        }
        ```
        
    - constexpr 예시이다.
        
        ```cpp
        class Point {
        public:
        	constexpr Point(double xVal = 0, double yVal = 0) noexcept
        	: x(xVal), y(yVal)
        	{}
        	
        	constexpr double xValue() const noexcept { return x; }
        	constexpr double yValue() const noexcept { return y; }
        	
        	constexpr void setX(double newX) noexcept { x = newX; } // C++14
        	constexpr void setY(double newY) noexcept { y = newY; } // C++14
        	
        private:
        	double x, y;
        };
        
        constexpr Point midpoint(const Point& p1, const Point& p2) noexcept
        {
        	return { (p1.xValue() + p2.xValue()) / 2,
        	         (p1.yValue() + p2.yValue()) / 2 };
        }
        
        constexpr Point p1(9.4, 27.6);
        constexpr Point p2(42.4, 12.6);
        
        constexpr auto mid = midpoint(p1, p2);
        ```
        
    - constexpr이 객체나 함수의 인터페이스의 일부이다. 이를 지정한다는 것은 이것이 상수 표현식을 요구하는 문맥에서 사용할 수 있다는 사실을 전하는 것이다. 나중에 constexpr을 제거하면 갑자기 컴파일이 되지 않는 코드가 생길지도 모르는 것이다. 이를 명심하고 적절히 사용하자.

- 항목 16 : const 멤버 함수를 스레드에 안전하게 작성하라
    - const 멤버 함수에서도 캐싱 등의 이유로 mutable 변수를 변경할 수 있다. 이럴 때, const 멤버 함수를 스레드에 안전하게 작성하게 작성해야 한다. 왜냐하면, 호출자(클라이언트)는 const 멤버 함수의 경우 당연히 스레드 안정성이 보장된다고 가정하고 사용할 여지가 크기 때문이다.
    - 물론, 동시적 문맥에서쓰이지 않을 것이 확실한 경우라면 스레드 안전성을 고려하지 않고 비용을 최소화할 수 있을 것이다.
    - std::atomic과 std::mutex를 적절히 사용하자. std::atomic이 std::mutex보다 비용이 싸다는 이유로 현혹되면 잘못 쓰는 경우가 생길 수 있다. 동기화가 필요한 변수 하나 또는 메모리 장소 하나에 대해서는 std::atomic을 사용하는 것이 적합하지만, 둘 이상의 변수나 메모리 장소를 하나의 단위로서 조작해야 할 때에는 뮤텍스를 꺼내는 것이 바람직하다.

- 항목 17 : 특수 멤버 함수들의 자동 작성 조건을 숙지하라
    - 특수 멤버 함수는 C++이 스스로 작성하는 멤버 함수들을 말한다. C++98에서는 기본 생성자, 소멸자, 복사 생성자, 복사 대입 연산자가 있었고, C++11에서는 이동 생성자, 이동 대입 연산자가 추가됐다.
        
        ```cpp
        class Widget {
        public:
        	Widget(Widget&& rhs); // 이동 생성자
        	Widget& operator=(Widget&& rhs); // 이동 대입 연산자
        };
        ```
        
    - 작성된 특수 멤버 함수들은 public이자 inline이며, 가상 소멸자가 있는 기본 클래스를 상속하는 파생 클래스의 소멸자를 제외하고는 비가상이다.
    - 이동 연산들은 필요할 때에만 작성되며, 클래스의 비정적 자료 멤버들에 대해 “**멤버별 이동**”을 수행한다. 또한, **기본 클래스 부분을 이동**한다. 만약 이동 연산을 지원하지 않는 경우엔 복사 연산을 수행한다. (이동 시, std::move를 사용한다)
    - 두 복사 연산은 서로 독립적이다. 하나를 선언한다고 해서 다른 하나의 자동 작성이 방지되지 않는다. 두 이동 연산은 독립적이지 않다. 둘 중 하나를 선언하면 컴파일러는 다른 하나를 작성하지 않는다.
    - 복사 연산을 하나라도 명시하면 컴파일러는 이동 연산들을 작성하지 않는다. 그 반대도 마찬가지다. 이동 연산을 하나라도 명시하면 컴파일러는 복사 연산들을 비활성화한다. 정확히는 delete한다.
    - 사용자 선언 소멸자가 있는 클래스에 대해서는 이동 연산들을 작성하지 않는다.
    - C++11에서는 복사 연산이나 소멸자를 선언하는 클래스에 대한 복사 연산들의 자동 작성이 비권장 기능으로 분류됐다. 따라서 자동으로 작성되는 것에 의존하는 클래스는 “= default”를 이용해서 의존성을 없애자.
    - 가상 소멸자도 default로 만들 수 있다. 일단 소멸자를 만들면, 이동 연산들을 자동으로 작성하지 않으니 만약 필요하다면 default로 만들어주자.
    - 그냥 가능한 자동 작성에 의존하지 말고 default로 명시하는 습관을 가지자.
    - 멤버 함수 템플릿 때문에 특수 멤버 함수의 자동 작성을 안 하는 경우는 전혀 없다. 아래와 같은 경우에도 Widget의 특수 멤버 함수는 자동 작성된다.
        
        ```cpp
        class Widget {
        template<typename T>
        Widget(const T& rhs);
        
        template<typename T>
        Widget& operator=(const T& rhs);
        };
        ```

