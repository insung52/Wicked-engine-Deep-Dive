# 커밋 #4: 359ade8d - Optimizations for hair particle system (Move Semantics & Const)

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `359ade8d` |
| 날짜 | 2025-11-14 |
| 작성자 | Turánszki János |
| 카테고리 | 최적화 / 코드 품질 |
| 우선순위 | 낮음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiAllocator.h | Move semantics 최적화, const 정확성 |
| (+ hair particle 관련 파일들) | 메인 기능 변경 |

**참고**: 커밋 제목은 "hair particle system"이지만, Allocator 변경은 DX12/Graphics 전반에 영향을 미치는 **부수적 최적화**입니다.

---

## 배경 지식: Move Semantics (이동 의미론)

### Copy vs Move

```
┌─────────────────────────────────────────────────────────────────┐
│                    Copy vs Move 비교                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Copy (복사):                                                   │
│  ┌─────────┐        ┌─────────┐                                 │
│  │ Source  │ ────→  │  Copy   │                                 │
│  │ (data)  │        │ (data)  │  ← 새 메모리 할당 + 복사        │
│  └─────────┘        └─────────┘                                 │
│       ↓                  ↓                                      │
│  [메모리 A]         [메모리 B]   ← 두 개의 메모리 사용           │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Move (이동):                                                   │
│  ┌─────────┐        ┌─────────┐                                 │
│  │ Source  │ ═══→   │  Dest   │                                 │
│  │ (data)  │        │ (data)  │  ← 포인터만 이동                │
│  └─────────┘        └─────────┘                                 │
│       ↓                  ↓                                      │
│  [nullptr]          [메모리 A]   ← 한 개의 메모리만 사용         │
│                                                                 │
│  이동 후 Source는 "빈" 상태 (valid but unspecified)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 언제 Move가 발생하는가?

```cpp
// 1. std::move() 명시적 사용
std::vector<int> a = {1, 2, 3};
std::vector<int> b = std::move(a);  // a의 내용이 b로 이동

// 2. 임시 객체 (rvalue)
std::vector<int> getVector() { return {1, 2, 3}; }
std::vector<int> c = getVector();  // 임시 객체 → 자동 이동

// 3. 함수 반환
std::vector<int> d = std::move(someVector);  // 명시적 이동
```

### Move 생성자와 Move 대입 연산자

```cpp
class MyClass {
    int* data;
    size_t size;

public:
    // Move 생성자
    MyClass(MyClass&& other) noexcept
        : data(other.data), size(other.size)
    {
        other.data = nullptr;  // 원본 비우기
        other.size = 0;
    }

    // Move 대입 연산자
    MyClass& operator=(MyClass&& other) noexcept
    {
        if (this != &other) {
            delete[] data;  // 기존 리소스 해제
            data = other.data;
            size = other.size;
            other.data = nullptr;  // 원본 비우기
            other.size = 0;
        }
        return *this;
    }
};
```

---

## 문제 상황: std::move 누락

### 잘못된 Move 구현

```cpp
// 문제: Move 생성자에서 std::move 없이 복사됨
Allocation(Allocation&& other)
{
    allocator = other.allocator;      // ⚠️ 복사! (other는 lvalue)
    internal_state = other.internal_state;  // ⚠️ 복사!
    // ...
}
```

### 왜 문제인가?

```
┌─────────────────────────────────────────────────────────────────┐
│                std::move 없는 "Move" 생성자                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Allocation(Allocation&& other)                                 │
│  {                                                              │
│      // other는 && 타입이지만, 이름이 있으므로 lvalue!          │
│      allocator = other.allocator;  // ← 복사 대입 호출          │
│  }                                                              │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  C++ 규칙:                                                      │
│  - 이름이 있는 것 = lvalue (좌측값)                             │
│  - 이름이 없는 것 = rvalue (우측값)                             │
│                                                                 │
│  other는 이름이 있으므로:                                       │
│  - other.allocator → lvalue                                     │
│  - allocator = other.allocator → 복사 대입 호출!               │
│                                                                 │
│  해결:                                                          │
│  - std::move(other.allocator) → rvalue로 캐스팅                │
│  - allocator = std::move(other.allocator) → 이동 대입 호출     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 실제 영향

```cpp
// shared_ptr의 경우:
std::shared_ptr<T> ptr1 = make_shared<T>();

// Move 없이 복사:
std::shared_ptr<T> ptr2 = ptr1;  // refcount 증가 (2)

// Move로 이동:
std::shared_ptr<T> ptr3 = std::move(ptr1);  // refcount 유지 (1), ptr1 = nullptr
```

| 연산 | refcount 변화 | 성능 |
|------|---------------|------|
| 복사 | +1 (atomic increment) | 느림 |
| 이동 | 0 (포인터만 복사) | 빠름 |

---

## 해결: std::move 추가

### 수정된 코드

```cpp
// Move 생성자
Allocation(Allocation&& other)
{
    Reset();
    allocator = std::move(other.allocator);      // ✅ 이동!
    internal_state = std::move(other.internal_state);  // ✅ 이동!
    byte_offset = other.byte_offset;
    other.allocator.reset();
    other.internal_state = nullptr;
    other.byte_offset = ~0ull;
}

// Move 대입 연산자
Allocation& operator=(Allocation&& other) noexcept
{
    Reset();
    allocator = std::move(other.allocator);      // ✅ 이동!
    internal_state = std::move(other.internal_state);  // ✅ 이동!
    byte_offset = other.byte_offset;
    other.allocator.reset();
    other.internal_state = nullptr;
    other.byte_offset = ~0ull;
    return *this;
}
```

---

## 배경 지식: const 정확성 (Const Correctness)

### const 멤버 함수

```cpp
class Container {
    std::vector<int> data;

public:
    // ❌ 잘못됨: 상태를 변경하지 않는데 const 없음
    bool is_empty() {
        return data.empty();
    }

    // ✅ 올바름: 상태를 변경하지 않으므로 const
    bool is_empty() const {
        return data.empty();
    }
};
```

### 왜 const가 중요한가?

```cpp
void processContainer(const Container& c) {
    // c는 const 참조
    if (c.is_empty()) {  // ⚠️ is_empty()가 const가 아니면 컴파일 에러!
        // ...
    }
}
```

| const 없음 | const 있음 |
|------------|------------|
| const 객체에서 호출 불가 | const 객체에서 호출 가능 |
| 최적화 기회 감소 | 컴파일러 최적화 가능 |
| API 의도 불명확 | "이 함수는 상태를 바꾸지 않음" 명시 |

---

## 문제: const 누락

### 수정 전

```cpp
struct BlockAllocator {
    // ⚠️ const 누락 - 상태를 변경하지 않는 함수
    inline bool is_empty()
    {
        return (blocks.size() * block_size) == free_list.size();
    }
};

struct PageAllocator {
    // ⚠️ const 누락
    inline bool is_empty()
    {
        return allocator->allocator.storageReport().totalFreeSpace == page_count;
    }
};
```

### 수정 후

```cpp
struct BlockAllocator {
    // ✅ const 추가
    inline bool is_empty() const
    {
        return (blocks.size() * block_size) == free_list.size();
    }
};

struct PageAllocator {
    // ✅ const 추가
    inline bool is_empty() const
    {
        return allocator->allocator.storageReport().totalFreeSpace == page_count;
    }
};
```

---

## VizMotive 적용 현황

### 이미 적용됨 ✅

**Allocator.h**:

| 위치 | 내용 |
|------|------|
| 61행 | `inline bool is_empty() const` (BlockAllocator) |
| 228행 | `inline bool is_empty() const` (PageAllocator) |
| 148-149행 | `std::move(other.allocator)`, `std::move(other.internal_state)` |
| 175-176행 | Move 대입 연산자에서 `std::move` 사용 |
| 290행 | `shared_ptr& operator=(shared_ptr&& other) noexcept` |
| 345행 | `weak_ptr& operator=(weak_ptr&& other) noexcept` |

---

## 성능 영향

### Move 최적화 효과

```
┌─────────────────────────────────────────────────────────────────┐
│                    Move 최적화 효과                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  shared_ptr/weak_ptr 이동 시:                                   │
│                                                                 │
│  Copy (수정 전):                                                │
│  - atomic refcount increment (+1)                               │
│  - 소멸 시 atomic refcount decrement (-1)                       │
│  - 메모리 배리어 2회                                            │
│                                                                 │
│  Move (수정 후):                                                │
│  - 포인터 복사만 (8바이트)                                      │
│  - atomic 연산 없음                                             │
│  - 메모리 배리어 없음                                           │
│                                                                 │
│  성능 차이: 10~100배 빠름 (atomic 연산 회피)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Const 최적화 효과

- 컴파일러가 함수가 상태를 변경하지 않음을 알 수 있음
- 더 나은 인라이닝 결정
- 더 나은 캐싱 최적화

---

## 요약

| 항목 | 내용 |
|------|------|
| 변경 1 | Move 생성자/대입에서 `std::move` 사용 |
| 효과 1 | 불필요한 복사 방지, atomic 연산 회피 |
| 변경 2 | `is_empty()`에 `const` 추가 |
| 효과 2 | const 객체에서 호출 가능, 최적화 기회 |
| VizMotive | ✅ 이미 적용됨 |

### 핵심 교훈

> **Move 생성자/대입에서는 반드시 std::move 사용**
>
> `&&` 파라미터라도 이름이 있으면 lvalue로 취급됨.
> 멤버를 이동하려면 `std::move(other.member)` 필수.

> **상태를 변경하지 않는 함수는 const로 선언**
>
> const 정확성은 코드의 의도를 명확히 하고,
> 컴파일러 최적화와 const 참조 사용을 가능하게 함.
