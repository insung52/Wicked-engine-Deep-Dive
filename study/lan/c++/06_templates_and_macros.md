# 템플릿과 매크로 (Templates vs Java Generics & Macros)

C++ 엔진 코드에는 꺾쇠 괄호 `<T>` (템플릿)와 `#` 으로 시작하는 전처리기 지시어 (매크로)가 매우 많다.
Java의 에서는 제네릭스를 쓰지만, 동작 방식이 C++의 템플릿과 근본적으로 다르다. 또한 Java에는 전처리기 매크로가 아예 존재하지 않는다.

---

## C++ 템플릿 vs Java 제네릭스

### 1. 타입 소거(Type Erasure) vs 코드 생성 (Monomorphization)

**Java의 제네릭 (Type Erasure)**
```java
List<String> list = new ArrayList<>();
List<Integer> list2 = new ArrayList<>();
```
컴파일할 때 타입 검사만 하고, 실제 런타임 바이트코드에는 모두 `List` (즉 `Object` 기반) 한 벌만 존재한다. 컴파일러가 자동 형변환을 넣어줄 뿐이다. `int` 같은 원시 타입(primitive)은 그대로 쓸 수 없어 `Integer` 래퍼(Wrapper) 클래스가 필요하고, 제네릭 타입 `<T>`로 `new T()` 등을 할 수 없다.

**C++의 템플릿 (Monomorphization)**
```cpp
std::vector<std::string> list;
std::vector<int> list2;
```
C++은 템플릿을 **"코드를 찍어내는 틀"**로 본다. `std::string`용 `vector` 클래스와 `int`용 `vector` 클래스의 소스코드를 컴파일러가 실제로 두 개 복사해서 각각 컴파일한다.
- `int` 같은 원시 타입도 박싱/언박싱 없이 최고 성능으로 사용 가능.
- 사용한 타입의 개수만큼 코드가 증식하므로 컴파일 시간이 길어지고 실행 파일 크기가 커질 수 있다.
- 런타임 비용은 제로(Zero-overhead).

### 2. Non-Type Template Parameter (값도 템플릿 인자가 될 수 있다)

Java는 타입(클래스/인터페이스)만 `<T>`로 넘길 수 있지만, C++은 **컴파일 타임 상수(정수 등)도 템플릿 인자**로 넘길 수 있다.

```cpp
template<typename T, int SIZE>
struct Array {
    T data[SIZE];
};

Array<int, 5> arr1;      // 길이가 5인 int 배열 (힙 할당 안 함)
Array<float, 10> arr2;   // 길이가 10인 float 배열
```
이 기능 때문에 C++ 코드를 보다 보면 `<T>` 안에 값 이 들어가는 코드를 자주 볼 수 있다.

---

## 전처리기 매크로 (Processor Macros)

컴파일 전에 코드의 텍스트를 치환하는 기능이다. Java에는 아예 없는 기능.
가장 많이 보이는 것은 `#define`, `#ifdef`, `#ifndef` 이다.

### 1. 상수 정의와 조건부 컴파일 (`#define`, `#ifdef`)

```cpp
#define DEBUG_MODE 1

#ifdef DEBUG_MODE
    // DEBUG_MODE가 정의되어 있을 때만 이 코드가 컴파일에 포함됨
    std::cout << "Debug info";
#else
    // 아니면 이 코드가 포함됨
#endif
```
VizMotive나 WickedEngine에서는 특정 플랫폼(Windows vs Linux)이나 그래픽 엔진 백엔드(DX12 vs Vulkan)를 선택할 때 이 매크로 블록을 수시로 사용하여 코드를 분리한다.

### 2. 매크로 함수

진짜 함수가 아니라 **텍스트 복사 붙여넣기**를 해주는 매크로다. 함수 호출 오버헤드가 없지만, 디버깅이 힘들고 예상치 못한 에러를 낼 수 있어 최근에는 `inline` 함수 도입으로 줄어드는 추세지만, 로깅 시스템 등에는 여전히 애용된다.

```cpp
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int x = MAX(10, 20); // 컴파일 시 int x = ((10) > (20) ? (10) : (20)); 로 코드가 치환됨
```

### 엔진 코드 내 매크로 예시

VizMotive 엔진에는 DX12 HRESULT 에러 체크 같은 반복 코드를 매크로로 감싸 놓은 것이 많다:

```cpp
// 매크로 정의
#define dx12_check(x) { HRESULT hr = (x); if(FAILED(hr)) { throw std::exception("DX12 Error"); } }

// 실제 코드 사용처
dx12_check(device->CreateCommandAllocator(..., IID_PPV_ARGS(&allocator)));
// 컴파일 시 위 코드 텍스트 덩어리가 통째로 위의 { } 블록으로 치환되어 에러 체크 코드가 삽입됨!
```
이런 코드를 만나면 "함수"가 아니라 "에러 체킹 템플릿 문법 조각" 정도로 이해하면 된다.
