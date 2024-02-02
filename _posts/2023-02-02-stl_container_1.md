---
title: "[C/C++] C++ STL Container [1]"
excerpt: "Array, Vector, List"

categories:
  - C/C++
tags:
  - [C, C++, stl, container, array, vector, list]

permalink: /c-cpp/stl_container_1/

toc: true
toc_sticky: true

date: 2024-02-02 23:00:00
last_modified_at: 2024-02-02
---
<br>

# 컨테이너(Container)?
C++ STL 컨테이너는 같은 타입의 객체들을 저장하고 관리할 수 있게 한다. template을 사용하여 제네릭하게 구현되어 있기 때문에 특정한 자료형에 의존하지 않는다.<br>

컨테이너들의 동작 원리를 이해하는 것이 중요하다. 예를 들어 vector는 맨 앞에 삽입/삭제하는 것이 비효율적이기 때문에 push_front(), pop_front()는 지원하지 않는다.

# 시퀀스 컨테이너 : 데이터가 선형적으로 저장되는 컨테이너

## 1. 배열 (std::array)
>헤더파일 \<array>, namespace std<br>
>template <typename T, std::size_t Num> class array;

<br>

### 특징
- 고정된 크기의 배열을 담는 컨테이너.
- 크기가 고정이기 때문에 원소의 추가나 삭제가 불가능하다.
- 값들이 메모리 상에서 연속적으로 저장되어 있다.
- 인덱스를 통한 임의 접근(Random Access)이 가능하며 빠르다 O(1).
- 주소로 접근할 경우 *(arr1), *(arr1+1) 접근 안 되고, *(arr1.data()), *(arr1.data() + 1) 이렇게 접근할 수 있다.
 
<br>

### 사용

```cpp
#include<iostream>
#include<array>
using namespace std;

int main()
{
	array<int, 5> arr1 = { 1,2,3,4,5 };
	cout << arr1[2] << endl; // 결과: 3
	
	array<double, 3> arr2;
	cout << arr2[2] << endl; // 결과: 쓰레기 값

	array<int, 6> arr3 = {};
	for (int temp : arr3)
	{
		cout << temp << endl;	// 결과: 모두 0
	}

	array<int, 6> arr4 = {1};
	for (int temp : arr4)
	{
		cout << temp << endl; // 결과: 1, 0, 0, 0, 0, 0
	}

	cout << *(arr1.data()) << *(arr1.data() + 1) << endl; // 결과: 1, 2

	return 0;
}
```

<br>

## 2. 벡터(vector)
>헤더파일 \<vector>, namespace std<br>
>template \<typename T> class vector;

<br>

### 특징

**동적 배열**이다. 배열(array)과 비슷한데, 고정된 크기가 아니라는 차이가 있다. 

- 고정된 크기가 아니다. 원소의 추가나 삭제가 가능하다.
- 값들이 메모리 상에서 연속적으로 저장되어 있다.

- 중간 삽입/삭제
삽입이나 삭제 위치 뒤의 모든 데이터를 한 칸씩 뒤로 밀거나 앞으로 당겨야 하기 때문에 느리다.

- 처음/끝 삽입/삭제
처음은 중간과 마찬가지로 느리다. 끝은 밀거나 당기지 않아도 돼서 빠르다.

- 임의 접근(Random Access)이 가능하다.

<br>

### 동작 원리
1. (여유분을 두고) 메모리를 할당한다.<br>
2. 여유분까지 꽉 찼으면, 더 큰 메모리를 할당해서 이사한다.

<br>

size: 실제 사용 데이터 개수<br>
capacity: 여유분을 포함한 용량 개수

```cpp
#include<iostream>
#include<vector>
using namespace std;

int main()
{
	vector<int> v1;

	for (int i = 0; i < 1000; i++)
	{
		v1.push_back(10);
		cout << v1.size() << " " << v1.capacity() << endl;
	}

	return 0;
}
```

size는 1개씩 늘고, size가 capacity를 넘게 되면 capacity는 1.5배 늘어난다. (컴파일러에 따라 다를 수도)<br>

resize()를 통해 size를 (capacity도 함께 증가), reserve()를 통해 capacity를 미리 크게 만들 수 있다. **데이터 이사를 막을 수 있다.**

```cpp
#include<iostream>
#include<vector>
using namespace std;

int main()
{
	vector<int> v1;

	v1.resize(1000);
	cout << v1.size() << " " << v1.capacity() << endl; // 1000, 1000

	return 0;
}
```

근데 이렇게 하면 이미 size가 1000이라 push_back하면 1001 된다.

```cpp
#include<iostream>
#include<vector>
using namespace std;

int main()
{
	vector<int> v1;

	v1.reserve(1000);
	cout << v1.size() << " " << v1.capacity() << endl; // 0, 1000

	return 0;
}
```

capacity 늘어난 상태에서 size 줄어들어도 capacity는 자동으로 줄어들지 않는다. reserve()로도 줄어들지 않는다. (swap하는 방법으로 capacity 낮추는 방법이 있긴 하다.)

<br>

### 사용
vector<int> v1(10, 1); // 모든 값이 1인 size = 10 벡터 생성

<br>

## 3. 리스트(list)
>헤더파일 \<list>, namespace std<br>
>template \<typename T> class list;

### 특징
- 노드 기반 컨테이너다.

<br>

### 동작 원리
**이중 연결 리스트**. 데이터가 메모리 상에 비연속적으로 저장된다. <br>

- 중간 삽입/삭제
이전 노드와 다음 노드를 가리키는 포인터들만 바꿔주면 되니깐 빠름. **근데 사실상 임의 접근이 안 되므로 그 위치로 가는 게 느림.**

- 처음/끝 삽입/삭제
중간 삽입/삭제와 동일.

- 임의 접근
임의 접근 불가능하다.

### 사용
- 벡터랑 다르게 삽입/삭제가 효율적이다 보니 특정 값을 가지는 데이터를 없애는 remove()가 있다.<br>
li.remove(10); // 10 모두 제거
