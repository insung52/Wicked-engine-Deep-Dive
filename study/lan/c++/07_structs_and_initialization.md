# 구조체, 클래스와 초기화 (Structs vs Classes & Initialization)

Java와 C++의 클래스는 비슷해 보이지만, **메모리에 저장되는 방식과 복사(Copy) 동작**이 완전히 다르다. 이 차이를 모르면 C++ 코드의 성능 문제나 버그 원인을 파악하기 힘들다.

---

## 1. Struct와 Class의 차이 (접근 제어자)

Java는 `class` 하나만 사용하지만, C++은 `struct`와 `class`가 사실상 동일한 기능을 하며 **전부 객체지향 (메소드, 상속, 소멸자 등 다 가능)** 으로 쓸 수 있다.
유일한 차이는 **기본 접근 제어자(Default Access Modifier)** 이다.

```cpp
// 1. struct: 기본값이 public
struct Data {
    int x; // public (외부에서 접근 가능)
private:
    int y; // 명시적으로 private 선언
};

// 2. class: 기본값이 private
class Actor {
    int x; // private (외부에서 접근 불가)
public:
    int y; // 명시적으로 public 선언
};
```
VizMotive/WickedEngine 코드에서는 단순히 **데이터(Data) 묶음을 담는 용도면 `struct`를 사용**하고, (예: `TextureDesc`, `CommandList_DX12`)
**기능(Method) 위주의 캡슐화가 중요한 용도면 `class`를 사용**한다 (예: `GraphicsDevice_DX12`). 관습적인 역할 분담일 뿐 문법적 기능 차이는 없다.

---

## 2. 기본 동작: 레퍼런스(Reference) vs 값 복사(Value Copy) — 가장 큰 차이

**Java는 모든 객체가 레퍼런스(Reference)다.**
```java
// Java
Vector2 a = new Vector2(10, 10);
Vector2 b = a; // 메모리상의 같은 객체를 둘이서 가리킴
b.x = 99;      // a.x 도 99가 됨!
```

**C++은 모든 객체가 기본적으로 값(Value)이다.**
```cpp
// C++
Vector2 a = {10, 10};  // 스택 메모리에 변수 생성
Vector2 b = a;         // ★ 메모리상의 a를 통째로 복사해서 스택에 다른 b를 만듦 (Deep Copy)
b.x = 99;              // a.x 는 여전히 10!
```
C++에서 Java처럼 `b`가 `a`의 내부를 가리키게 만들려면 **포인터(`*`)**나 **참조(`&`)**를 명시적으로 써야 한다.

```cpp
Vector2 a = {10, 10};
Vector2& b = a; // b는 a의 별명(Reference)
b.x = 99;       // a.x 도 99가 됨!
```

이 규칙은 함수 파라미터(인자)를 전달할 때도 동일하게 적용된다. C++에서 무심코 `void DoSomething(TextureDesc desc)` 라고 코딩하면, 호출할 때마다 거대한 구조체 메모리 전체를 복사해서 넘기게 된다! 그래서 **`const Type&`** 패턴이 매우 중요하다. (01_pointers_and_references.md 참고)

---

## 3. 유니폼 초기화 (Uniform Initialization)

Java는 객체를 생성할 때 생성자를 호출한다(`new Obj()`). C++은 중괄호 `{}`를 사용한 초기화 문법이 존재한다. 빠르고 직관적이다. 대입 연산자 `=`를 생략하고 바로 쓰는 경우가 많다.

```cpp
// 기본 구조체
struct Point { int x, y; };

Point p1 = {10, 20}; // 배열처럼 초기화
Point p2{10, 20};    // 더 최신 C++ 문법 (= 생략 가능)

// 클래스 객체 (생성자 인자로 전달)
std::vector<int> arr{1, 2, 3}; // 인자 3개 전달이 아닌 [1, 2, 3] 벡터 생성

// 함수 리턴 시 타입 생략 가능
Point getOrigin() {
    return {0, 0}; // 알아서 Point{0,0}으로 추론됨
}
```
WickedEngine의 구조체 세팅 코드에서 이런 형태가 아주 많이 보인다.
```cpp
// VizMotive의 실제 초기화 예시
TextureDesc desc;
desc.width = 1920;
desc.height = 1080;
desc.format = Format::R8G8B8A8_UNORM;
// C++11 이상에서는 위를 다음과 같이 한 줄로 요약 가능:
TextureDesc desc{ 1920, 1080, ... }; 
```

---

## 4. `auto` 키워드 (Java의 `var`와 동일)

컴파일러가 초기화 값을 보고 변수 타입을 알아서 추론한다.

```cpp
auto x = 10;        // x는 int (10이 int니까)
auto f = 3.14f;     // f는 float
auto* ptr = &x;     // ptr은 int* (auto ptr = &x; 도 가능)

// 긴 타입을 줄여주어 매우 유용.
// std::unordered_map<std::string, std::shared_ptr<Texture>>::iterator it = map.begin();
auto it = map.begin(); 
```
단, `auto`는 **항상 값 복사(Value Copy)** 로 추론된다. 원본을 참조로 건드리고 싶으면 무조건 **`auto&`** 를 적어야 함에 주의! (이 또한 Java와 전혀 다른 점이다).

```cpp
std::vector<std::string> words = {"apple", "banana"};

for(auto w : words) { 
    // w는 값 복사! 문자열 할당이 매번 일어남 (느림)
}

for(const auto& w : words) {
    // w는 읽기 전용 참조! 문자열 복사가 전혀 안 일어남 (빠르고 안전함. C++ 루프의 표준 방식)
}
```
