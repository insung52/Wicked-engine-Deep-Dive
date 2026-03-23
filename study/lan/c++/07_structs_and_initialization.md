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

---

## 5. 멤버 접근 연산자 (`.`, `->`, `::`)

Java는 객체의 멤버(변수, 함수)에 접근할 때 마침표(`.`) 하나만 쓰지만, C++은 메모리 접근 방식(값 vs 포인터)이 중요하므로 상황에 따라 세 가지 연산자를 명확히 구분해서 사용해야 한다.

1. **`.` (Dot Operator, 점 연산자)**: **객체(값) 자체** 또는 **레퍼런스(Reference)** 변수에서 멤버를 꺼낼 때 사용한다.
2. **`->` (Arrow Operator, 화살표 연산자)**: **포인터(Pointer)** 가 가리키는 메모리 주소를 따라가서 멤버를 꺼낼 때 사용한다. (사실상 `(*ptr).member`의 축약형이다.)
3. **`::` (Scope Resolution Operator, 범위 지정 연산자)**: 어떤 **소속**인지 명시할 때 쓴다. 구조체/클래스(형틀) 자체에 속한 **정적(static) 멤버**, **중첩 타입(enum 등)**, 또는 **네임스페이스** 안으로 접근할 때 사용한다.

```cpp
struct Player {
    int hp;
    void Move();
    static int totalPlayers; // 모든 Player 인스턴스가 공유하는 정적 변수
};

// 정적 변수는 클래스 외부에서 메모리 할당(정의)이 필요함 (:: 사용)
int Player::totalPlayers = 0; 

void Test() {
    Player p1;
    p1.hp = 100;    // 1. 값(객체) 직접 접근 (.)
    p1.Move();      // .

    Player* ptr = &p1;
    ptr->hp = 50;   // 2. 포인터 주소를 따라가서 접근 (->)
    ptr->Move();    // ->

    int count = Player::totalPlayers; // 3. 클래스 소속 정적 요소 접근 (::)
}
```

---

## 6. `this` 포인터와 정적(static) 멤버

### `this`는 자바처럼 레퍼런스가 아니라 '포인터'다
Java에서의 `this`는 현재 객체를 가리키는 숨겨진 키워드(레퍼런스)라 `this.hp`처럼 점(`.`)을 찍습니다.
하지만 **C++에서의 `this`는 현재 객체의 메모리 주소를 담고 있는 숨겨진 '포인터'** 입니다.
따라서 `this`를 사용해 멤버에 접근할 때는 반드시 화살표(`->`) 연산자를 써야 합니다.

```cpp
struct Monster {
    int hp;
    
    void SetHP(int hp) {
        // 매개변수 이름(hp)과 멤버 변수 이름(hp)이 같을 때 구분하는 방법
        this->hp = hp; // this가 포인터이므로 -> 사용 (Java의 this.hp 가 아님!)
    }
    
    Monster* GetPointer() {
        return this; // 자기 자신의 메모리 주소를 반환할 때도 this 사용
    }
};
```

### 정적(static) 멤버 함수와 `this`
클래스/구조체 내부에 선언된 `static` 함수는 특정 객체(Instance) 메모리에 묶여있지 않고, 프로그램 전체에서 클래스 이름 공간에 하나만 존재합니다.

따라서 **`static` 함수 내부에는 `this` 포인터가 아예 존재하지 않습니다.**
결과적으로 C++의 정적 함수 안에서는 객체 고유의 일반 멤버 변수(hp 등)를 직접 읽거나 쓸 수 없으며, 오직 다른 `static` 멤버 변수나 매개변수로 넘어온 데이터만 다룰 수 있습니다.

```cpp
struct SystemManager {
    int normal_var = 10;
    static int static_var;

    static void DoWork() {
        static_var = 20;    // (O) 정적 멤버 접근 가능
        // normal_var = 20; // (X) 컴파일 에러! 어떤 객체의 normal_var인지 알 수 없음 (this 없음)
    }
};
int SystemManager::static_var = 0; // 초기화
```

---

## 7. 메모리 정렬 (Memory Alignment)

### 개념: SIZE와 ALIGNMENT는 별개다

**SIZE** = 데이터가 차지하는 바이트 수
**ALIGNMENT** = 데이터가 시작해야 하는 주소의 제약 (N의 배수)

이 둘은 완전히 독립적이다.

```cpp
struct Example {
    char   c;   // size=1, alignment=1
    double d;   // size=8, alignment=8  ← 주소가 반드시 8의 배수여야 함
    int    i;   // size=4, alignment=4
};
// sizeof(Example) = 24  (padding 때문에 1+7+8+4+4 = 24)
```

### 왜 alignment가 필요한가

CPU는 메모리를 1바이트씩 읽지 않는다. **하드웨어 버스 너비 단위**(4바이트, 8바이트 등)로 읽는다.
만약 `double`이 주소 1에서 시작하면 CPU는 2번 읽어서 조합해야 한다 (느림 + 일부 아키텍처는 크래시).
그래서 컴파일러가 자동으로 **패딩(padding)** 바이트를 삽입해 정렬을 맞춰준다.

```
주소: 0   1   2   3   4   5   6   7   8   9  10  11
     [c] [pad pad pad pad pad pad pad] [d d d d d d d d] ...
      ↑                               ↑
      char(1B)     padding(7B)        double(8B) — 주소 8에서 시작 = 8의 배수 ✓
```

### struct 멤버 순서로 크기가 달라진다

```cpp
struct Bad {   // 24바이트
    char   a;  // 1B + padding 7B
    double b;  // 8B
    char   c;  // 1B + padding 7B
    double d;  // 8B
};

struct Good {  // 18바이트 → 실제로는 정렬 때문에 24바이트가 아닌 20바이트
    double b;  // 8B (먼저 배치)
    double d;  // 8B
    char   a;  // 1B
    char   c;  // 1B + padding 2B (int 정렬 기준 맞추기)
};
// 규칙: 큰 멤버 먼저 배치하면 padding 낭비가 줄어듦
```

실제 계산보다는 **"큰 것 먼저, 작은 것 나중에"** 원칙을 기억하는 게 실용적이다.

### `alignas` 키워드

컴파일러 기본 정렬보다 더 강한 제약을 명시적으로 줄 때 사용한다.

```cpp
// 구조체 전체에 적용: 시작 주소를 256의 배수로 강제
struct alignas(256) ConstantBuffer {
    float4 worldMatrix;
    float4 viewMatrix;
};
// → 이 구조체를 배열로 놓으면 각 원소가 256바이트 경계에 정렬됨

// 멤버 하나에 적용:
struct alignas(16) Vec4 {
    float x, y, z, w;
};
```

**VizMotive 실제 사용 예:**
`Allocator.h`의 `SharedHeapAllocator::RawStruct`에 `alignas(256)` 적용.
GPU 상수 버퍼는 DX12 요구사항에 따라 256바이트 경계에 배치되어야 하기 때문이다.

### `sizeof` vs `alignof`

```cpp
struct alignas(16) Vec4 { float x, y, z, w; };

sizeof(Vec4)  // 16 — 데이터 크기
alignof(Vec4) // 16 — 정렬 요구사항

// sizeof는 항상 alignof의 배수다 (패딩 포함하기 때문)
```

---

## 8. 상속, virtual, override, final

### 기본 상속

```cpp
struct Base {
    int x;
    void foo() { /* Base의 foo */ }
};

struct Derived : Base {
    int y;
    void foo() { /* Derived의 foo — Base::foo를 가린다(hiding) */ }
};
```

- `Derived`는 `Base`의 멤버(`x`, `foo`)를 모두 상속받는다.
- 그러나 `Derived::foo()`는 `Base::foo()`를 **오버라이드한 게 아니다** — 이름만 같은 별개 함수(name hiding).

### virtual — 오버라이드를 가능하게 하는 키워드

`virtual`이 없으면 **정적 바인딩(컴파일 시 결정)**: 포인터 타입 기준으로 함수가 호출된다.
`virtual`이 있으면 **동적 바인딩(런타임 결정)**: 실제 객체 타입 기준으로 함수가 호출된다.

```cpp
struct Base {
    virtual void foo() { /* Base */ }  // virtual 선언
};

struct Derived : Base {
    void foo() override { /* Derived */ }  // override 키워드: 의도 명시
};

Base* ptr = new Derived();
ptr->foo();  // → Derived::foo() 호출 (동적 바인딩)
```

### non-virtual 함수는 오버라이드 불가

```cpp
struct Base {
    void bar() { /* Base */ }  // non-virtual
};

struct Derived : Base {
    void bar() { /* Derived */ }  // name hiding (오버라이드 아님!)
};

Base* ptr = new Derived();
ptr->bar();  // → Base::bar() 호출! (정적 바인딩)
```

**VizMotive 실제 사례:** `Resource_DX12::destroy_subresources()`가 non-virtual이므로,
`Texture_DX12`에서 같은 이름의 함수를 만들어도 `Base*` 포인터로 호출 시 `Resource_DX12` 버전이 호출된다.
→ `apply_texture.md` 참고

### vptr과 vtable — virtual의 비용

`virtual` 함수가 있는 클래스의 모든 인스턴스는 **vptr(가상 함수 테이블 포인터)** 8바이트를 갖는다.
vptr은 vtable(함수 포인터 배열)을 가리킨다. 런타임에 vtable을 통해 실제 함수를 찾는다.

```
Resource_DX12 인스턴스 메모리:
[vptr(8B)] [멤버들...]
    ↓
  vtable: [ &Resource_DX12::destroy_subresources, ... ]
```

GPU 리소스처럼 수천 개가 생성되는 객체에 불필요한 virtual을 추가하면 메모리/캐시 낭비가 된다.

### `override` 키워드 (C++11)

```cpp
struct Derived : Base {
    void foo() override;  // Base에 virtual foo()가 없으면 컴파일 에러
};
```

실수로 함수 시그니처가 달라서 오버라이드 실패하는 버그를 컴파일 시점에 잡아준다.

### `final` 키워드 (C++11)

```cpp
// 클래스에 final: 더 이상 상속 불가
struct Texture_DX12 final : Resource_DX12 { ... };

// 함수에 final: 이 함수를 하위 클래스에서 오버라이드 불가
virtual void foo() final;
```

클래스에 `final`이 붙으면 컴파일러가 가상 함수 호출을 devirtualize(정적 바인딩으로 최적화)할 수 있다.

### 요약 표

| | non-virtual | virtual |
|---|---|---|
| 바인딩 시점 | 컴파일 타임 (정적) | 런타임 (동적) |
| 파생 클래스 재정의 | name hiding (오버라이드 아님) | `override`로 진짜 오버라이드 |
| vptr 비용 | 없음 | 인스턴스당 8바이트 |
| `Base*`로 호출 시 | Base 버전 호출 | 실제 타입 버전 호출 |
