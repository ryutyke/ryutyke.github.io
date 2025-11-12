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