---
title: "[C/C++] C++ STL Container [2]"
excerpt: "map, unordered_map"

categories:
  - C/C++
tags:
  - [C, C++, stl, container, map, unordered_map]

permalink: /c-cpp/stl_container_2/

toc: true
toc_sticky: true

date: 2024-02-03 17:00:00
last_modified_at: 2024-02-02
---
<br>

# 컨테이너(Container)?
C++ STL 컨테이너는 같은 타입의 객체들을 저장하고 관리할 수 있게 한다. template을 사용하여 제네릭하게 구현되어 있기 때문에 특정한 자료형에 의존하지 않는다.<br>

# 연관 컨테이너
vector, list와 같은 시퀀스 컨테이너는 원하는 조건에 해당하는 데이터를 빠르게 찾을 수 없다는 단점이 있다.
연관 컨테이너는 Key – Value 구조를 가지는 컨테이너로, 요소들에 대한 빠른 접근을 제공한다.

## 1. 맵 (map)
>헤더파일 \<map>, namespace std<br>
>template \<typename Key, typename Value> class map;

<br>

### 특징
- 노드 기반이다.

```cpp
class Node
{
public:
	Node* _left;
	Node* _right;
	pair<int, string> _data;
};
```

- data의 자료형이 pair라 pair로 삽입해야 한다.

```cpp
int main()
{
	map<int, int> m1;
	
	m1.insert(pair<int, int>(2, 5));
	m1.erase(2);

	return 0;
}
```

- Red-Black Tree(자가 균형 이진 탐색 트리)로 구현되어 있다. 때문에 삽입, 삭제, 검색 모두 O(logN). **최악의 경우에도 O(logN)는 지킴.**
- key 값 중복 안 되고, 데이터를 정렬하여 저장.
- 정렬을 유지하기 위한 추가적인 작업이 필요해서 원소의 삽입/삭제가 빈번할 경우 성능이 저하된다.

<br>

### 코드

```cpp
#include<iostream>
#include<map>
using namespace std;

int main()
{
	map<int, int> m1;
	
	for (int i = 0; i < 100; i++)
	{
		m1.insert(pair<int, int>(i, i*2));
	}

	m1.erase(2); // key가 2인 노드 제거
	
	map<int,int>::iterator findIt = m1.find(2); // key가 2인 노드 찾기 : 못 찾음
	if (findIt != m1.end())
	{
		cout << "찾음" << endl;
	}
	else { cout << "못 찾음" << endl; }

	findIt = m1.find(4); // key가 4인 노드 찾기 : 찾음
	if (findIt != m1.end())
	{
		cout << "찾음" << endl;
	}
	else { cout << "못 찾음" << endl; }

	for (map<int, int>::iterator it = m1.begin(); it != m1.end(); ++it)
	{
		pair<const int, int>& p = (*it);
		int key = it->first;
		int value = it->second;

		cout << key << " " << value << endl;
	}

	m1.insert(make_pair(1, 4)); // 이미 있던 key라면 value 수정 안 됨. return 해주는 pair.second가 false.

	// 없으면 추가, 있으면 수정해 주는 코드
	map<int, int>::iterator findIt2 = m1.find(1);
	if (findIt2 != m1.end())
	{
		findIt2->second = 4;
	}
	else
	{
		m1.insert(make_pair(1, 4));
	}

	// 사실 이건 이런 식으로 제공된다.
	m1[1] = 4;
	
	// 이렇게 해도 key = 1000인 값이 추가되므로 주의해야 한다..
	cout << m1[1000] << endl;

	return 0;
}
```

<br>

## 2. 해시맵 (unordered map)
>헤더파일 \<unordered_map>, namespace std<br>
>template \<typename Key, typename Value> class unordered_map;

<br>

### 특징

- Hash Table 기반 (map은 Red-black Tree)
- 값 중복 안 되고, 정렬되지 않고 랜덤하게 저장된다.
- insert, erase, find 모두가 O(1).
- (중요) 검색 시 **해시 충돌이 일어나서 O(N)까지도 걸릴 수 있다.** 안정적인 속도가 중요한 작업이라면 항상 O(logN)인 map을 사용하는 것이 좋을 수도 있다. 최적화가 중요하다면 **해시 함수를 잘 짜서** unordered map 사용.
- 직접 만든 클래스를 넣으려면 해시 함수를 직접 만들어야 한다. C++ STL에서 int, string 등 기본적인 타입들에 대한 해시 함수를 제공하기 때문에 이를 잘 활용해서 만들면 된다.
- Bucket을 늘려야 하는 경우에 해시 함수를 바꿔야 해서 기존 원소들을 모두 다시 insert 해줘야 하는, O(N)이 걸리는 rehash 작업이 실행된다. (약간 벡터 capacity 늘리는 느낌) 