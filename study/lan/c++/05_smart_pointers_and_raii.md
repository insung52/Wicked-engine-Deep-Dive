# 스마트 포인터와 RAII (Smart Pointers & RAII)

Java는 가비지 컬렉터(GC)가 주기적으로 메모리를 정리해 주지만, C++은 프로그래머가 메모리 수명을 직접 관리해야 한다.
이를 안전하게 하기 위해 C++에서는 **RAII** 패턴과 **스마트 포인터**를 사용한다.

---

## RAII (Resource Acquisition Is Initialization)

"자원 획득은 초기화다"라는 뜻인데, 실제 의미는 **"객체의 수명(scope)이 끝나면 소멸자(destructor)가 자동으로 호출되는 C++의 특징을 이용해 자원을 해제하자"**는 패턴이다.

Java의 `try-with-resources` 블록이나 `synchronized` 블록이 C++에서는 RAII를 통해 깔끔하게 구현된다.

### 1. 락(Lock)과 동기화 예시

Java에서는 블록을 사용한다:
```java
synchronized (mutex) {
    // 임계 구역
} // 블록이 끝나면 락 해제
```

C++에서는 `std::scoped_lock` (또는 `lock_guard`) 이라는 RAII 객체를 사용한다:
```cpp
std::mutex locker;

void doSomething() {
    std::scoped_lock lock(locker); // 생성자에서 lock 획득
    
    // 임계 구역 (critical section)
    if (error) return; // return 하거나 예외가 발생하더라도...
    
} // 함수(또는 블록)가 끝나면 lock 객체가 해제되며 소멸자에서 락을 해제함
```
생성자에서 락을 걸고, 소멸자에서 락을 푸는 객체를 스택에 만들면, 블록을 빠져나갈 때 알아서 락이 풀린다.

---

## 스마트 포인터 (Smart Pointers)

`new`로 메모리를 할당하고 `delete`를 까먹으면 메모리 누수(leak)가 발생한다.
이를 막기 위해 내부적으로 포인터를 감싸고, 자신의 수명이 끝날 때 자동으로 `delete`를 호출해주는 클래스들이 바로 스마트 포인터다.

### 1. 단독 소유 `std::unique_ptr`

어떤 포인터의 소유권이 "단 한 곳"에만 있을 때 사용. 가장 가볍고(raw 포인터와 동일한 크기 및 성능), 복사가 불가능하다.

```cpp
#include <memory>

void func() {
    // 힙에 데이터 객체 할당하고 unique_ptr로 감쌈 (C++14 이상 권장 방식)
    std::unique_ptr<Data> p = std::make_unique<Data>(10, 20);
    
    p->doSomething();
    
    // std::unique_ptr<Data> p2 = p; // 컴파일 에러! 복사 불가 (단독 소유니까)
    std::unique_ptr<Data> p2 = std::move(p); // 소유권 이전은 가능. 이제 p는 nullptr.
    
} // p2의 수명이 끝나면 자동으로 Data 객체의 소멸자가 호출되고 메모리 해제
```

### 2. 공동 소유 `std::shared_ptr`

여러 곳에서 참조해야 하는 객체에 사용. Java의 가비지 컬렉션 대상을 다루는 느낌과 가장 비슷하다.
내부적으로 **"참조 카운트(Reference Count)"**를 유지한다. 복사될 때마다 카운트가 오르고, 변수가 소멸할 때 카운트가 내려간다. 0이 되면 메모리를 해제한다.

```cpp
std::shared_ptr<Data> p1 = std::make_shared<Data>(); // 카운트 = 1

{
    std::shared_ptr<Data> p2 = p1; // 카운트 = 2 (p1, p2가 공유함)
} // p2가 블록을 벗어남 → 카운트 = 1 (삭제 안 됨)

// p1이 블록을 벗어남 → 카운트 = 0 → Data 메모리 자동 해제
```

#### make_shared와 control block

`shared_ptr`는 refcount를 **control block**이라는 별도 구조체에 저장해야 한다.

T(Data) 객체 안에 넣을 수 없는 이유: Data의 정의를 바꿀 수 없고, Data는 shared_ptr를 모른다.

`new Data()`를 shared_ptr로 감쌀 때 — 힙 할당이 **2번** 발생:
```cpp
std::shared_ptr<Data> p(new Data());  // Data 힙 할당 + control block 힙 할당
```
```
  힙 할당 ① → Data 객체
  힙 할당 ② → control block (refcount 보관)

  shared_ptr 내부:
  ┌──────────────┬──────────────┐
  │    Data* ptr │ ctrl_block*  │  ← 16바이트
  └──────────────┴──────────────┘
```

`make_shared<Data>()`는 Data + control block을 **한 번에** 힙 할당:
```cpp
std::shared_ptr<Data> p = std::make_shared<Data>();  // 힙 할당 1번
```
```
  힙 할당 ① (한 번에):
  ┌────────────────────────┐
  │ control block (refcount│
  ├────────────────────────┤
  │ Data 객체               │
  └────────────────────────┘

  shared_ptr 내부:
  ┌──────────────┬──────────────┐
  │    Data* ptr │ ctrl_block*  │  ← 둘 다 위 블록 안에 있음
  └──────────────┴──────────────┘
```

힙 할당 횟수를 1번으로 줄이지만, **여전히 힙 할당 자체는 발생한다**.
→ 매 프레임 수백 개의 GPU 리소스를 생성/삭제하는 그래픽스 엔진에서는 이게 부담이 된다.
→ 해결책이 Block Allocator 풀링 (힙 할당 없이 미리 준비된 슬롯 재사용).

> **참고**: VizMotive나 WickedEngine에서는 직접 구현한 커스텀 shared_ptr를 사용하기도 한다 (Block Allocator와 결합해 힙 할당을 제거하기 위함).

### 3. 상호 참조 끊기 `std::weak_ptr`

`shared_ptr` A가 B를 가리키고, B가 A를 가리키면 참조 카운트가 절대 0이 되지 않아 메모리 누수(순환 참조, Circular Reference)가 발생한다.
이럴 때 한쪽을 `weak_ptr`로 선언해, "가리키고는 있되 카운트는 올리지 않도록" 만든다. 사용할 때는 `lock()`을 이용해 확인 후 쓴다.

```cpp
shared_ptr<T> strong = make_shared<T>();  // refcount = 1
weak_ptr<T>   weak   = strong;            // refcount 그대로 1 (증가 없음)

strong.reset();  // refcount = 0 → ~T() 호출, 객체 파괴

// 나중에 weak에서 사용 시도:
shared_ptr<T> locked = weak.lock();
if (locked)
    // 객체 아직 살아있음
else
    // 객체 이미 파괴됨
```

#### lock()의 올바른 구현 — CAS

`lock()`의 핵심: "refcount가 0이 아닐 때만 원자적으로 증가"시켜야 한다.
단순히 inc → check → dec 순서로 하면 멀티스레드에서 race condition이 생긴다:

```
잘못된 구현:                        올바른 구현 (CAS):

inc_refcount()                      try_inc_refcount()
  refcount: 0 → 1                     do {
                                         if (refcount == 0) return false;
[다른 스레드: 객체 파괴 + 메모리 재사용]  } while (!CAS(expected, expected+1));
                                       → 체크와 증가를 원자 연산 1번으로 처리

check → refcount가 1이니 "살아있다" 판단  → 0이면 false 즉시 반환
dec_refcount()                          → 실패 시 아무것도 건드리지 않음
  refcount: 1 → 0
  → 다른 객체의 메모리를 잘못 조작!
```

CAS(Compare-And-Swap): "현재 값이 expected와 같으면 새 값으로 바꾸고 true, 아니면 false"를 **원자적으로** 수행하는 CPU 명령어.

---

## COM 스마트 포인터 `ComPtr<T>`

DirectX 프로그래밍(VizMotive 엔진 포함)에서는 마이크로소프트가 제공하는 COM 인터페이스(예: `ID3D12Device`, `ID3D12CommandQueue`)를 다룬다.
이 COM 객체들은 자체적인 참조 카운팅 시스템(`AddRef()`, `Release()`)을 가지고 있다.

이때 컴포넌트를 안전하게 관리하기 위해 DirectX 코드에서는 `std::shared_ptr` 대신 **`Microsoft::WRL::ComPtr`**를 사용한다.

```cpp
#include <wrl/client.h>
using Microsoft::WRL::ComPtr;

ComPtr<ID3D12CommandAllocator> allocator;

// 생성할 때
device->CreateCommandAllocator(..., IID_PPV_ARGS(&allocator)); 
// 내부 포인터 주소를 넘길 때 &연산자 사용 (주소의 주소)

// 사용할 때 (포인터처럼)
allocator->Reset();

// 변수가 스코프를 넘어가거나 클래스가 파괴될 때 자동으로 Release()를 호출해 줌.
```
VizMotive 엔진 내의 대부분의 DX12 리소스는 `ComPtr`로 선언되어 있어, 일일이 `Release()`를 호출하지 않아도 자동으로 메모리가 해제된다.

---

## 요약
- **단일 소유, 빠른 속도**: `std::unique_ptr`
- **공유 (Java 객체 느낌)**: `std::shared_ptr`
- **DirectX 리소스 (엔진 핵심)**: `ComPtr`
- **동기화 락 (Synchronized 대체)**: `std::scoped_lock` / `std::lock_guard`
