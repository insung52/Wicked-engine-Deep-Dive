# 커밋 #11: 90e94ed4 - Placement new is allowed for gfx objects

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `90e94ed4` |
| 날짜 | 2025-11-24 |
| 작성자 | Turánszki János |
| 카테고리 | 리팩토링 |
| 우선순위 | 낮음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphics.h | GPUBuffer, Texture, RaytracingAccelerationStructure에서 placement new 허용 |

---

## 배경: 커밋 #7에서의 동적 할당 금지

### 커밋 #7 복습

커밋 #7에서 vtable 제거와 함께 동적 할당을 금지했습니다:

```cpp
struct GPUBuffer final : public GPUResource {
    // 모든 동적 할당 금지
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;

    // placement new도 금지됨!
    static void* operator new(size_t, void*) = delete;
    static void* operator new[](size_t, void*) = delete;
};
```

### 왜 동적 할당을 금지했는가?

```
┌─────────────────────────────────────────────────────────────────┐
│              동적 할당 금지 이유 (커밋 #7)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. vtable이 없으므로 다형성(polymorphism) 불가                 │
│     ─────────────────────────────────────────                   │
│     GPUResource* ptr = new GPUBuffer();                         │
│     delete ptr;  // 💥 GPUResource 소멸자만 호출됨!             │
│                  //    GPUBuffer 소멸자 호출 안 됨              │
│                                                                 │
│     vtable이 있으면 가상 소멸자로 올바른 소멸자 호출            │
│     vtable이 없으면 베이스 클래스 소멸자만 호출                 │
│                                                                 │
│  2. 실수 방지                                                   │
│     ─────────────                                               │
│     Graphics 객체는 보통 스택/멤버 변수로 사용                  │
│     동적 할당을 막아서 잘못된 사용 패턴 컴파일 에러로 방지      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 문제: Placement New도 금지됨

### Placement New란?

```cpp
// 일반 new: 메모리 할당 + 객체 생성
GPUBuffer* buf = new GPUBuffer();  // 1. malloc(sizeof(GPUBuffer))
                                    // 2. GPUBuffer 생성자 호출

// Placement new: 이미 할당된 메모리에 객체 생성
void* memory = allocator.allocate(sizeof(GPUBuffer));  // 1. 메모리 할당 (별도)
GPUBuffer* buf = new (memory) GPUBuffer();             // 2. 생성자만 호출
```

### 왜 Placement New가 필요한가?

```
┌─────────────────────────────────────────────────────────────────┐
│              Placement New 사용 사례                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Block Allocator (커밋 #8)                                   │
│     ─────────────────────────                                   │
│     template<typename T>                                        │
│     struct BlockAllocator {                                     │
│         T* allocate() {                                         │
│             void* memory = free_list.pop();                     │
│             return new (memory) T();  // placement new 필요!    │
│         }                                                       │
│     };                                                          │
│                                                                 │
│  2. 메모리 풀링                                                 │
│     ─────────────                                               │
│     미리 할당된 메모리 블록에 객체들을 생성                     │
│     힙 할당 오버헤드 없이 객체 생성 가능                        │
│                                                                 │
│  3. 커스텀 메모리 할당자                                        │
│     ─────────────────────                                       │
│     특정 메모리 영역(GPU 메모리, 공유 메모리 등)에 객체 생성    │
│                                                                 │
│  4. std::vector 내부 구현                                       │
│     ─────────────────────                                       │
│     vector는 내부적으로 raw memory를 할당하고                   │
│     placement new로 객체를 생성함                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 커밋 #7의 과도한 제한

```cpp
// 커밋 #7 - 모든 new 금지 (placement new 포함)
static void* operator new(size_t, void*) = delete;      // ← 이것도 금지됨
static void* operator new[](size_t, void*) = delete;    // ← 이것도 금지됨

// 결과: Block Allocator에서 Graphics 객체 생성 불가!
// BlockAllocator<GPUBuffer> allocator;
// GPUBuffer* buf = allocator.allocate();  // 💥 컴파일 에러!
```

---

## 해결: Placement New만 허용

### 수정된 코드

```cpp
struct GPUBuffer final : public GPUResource {
    // 일반 동적 할당은 여전히 금지
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;

    // Placement new는 허용! (메모리 할당 없이 생성자만 호출)
    static void* operator new(size_t, void* p) noexcept { return p; }
    static void* operator new[](size_t, void* p) noexcept { return p; }
};
```

### Placement New 구현 분석

```cpp
static void* operator new(size_t, void* p) noexcept { return p; }
//                        ^^^^^   ^^^^^^
//                        |       |
//                        |       └─ 이미 할당된 메모리 주소
//                        └─ 필요한 크기 (사용 안 함)

// 동작:
// 1. 전달받은 포인터 p를 그대로 반환
// 2. 메모리 할당 없음 (이미 할당되어 있음)
// 3. 컴파일러가 이 주소에 생성자 호출
```

---

## 허용/금지 비교

```
┌─────────────────────────────────────────────────────────────────┐
│              New 연산자 허용/금지 현황                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  연산자                          │ 커밋 #7  │ 커밋 #11 │        │
│  ────────────────────────────────┼──────────┼──────────┤        │
│  new GPUBuffer()                 │ ❌ 금지  │ ❌ 금지  │ 힙 할당│
│  new GPUBuffer[10]               │ ❌ 금지  │ ❌ 금지  │ 힙 할당│
│  delete buf                      │ ❌ 금지  │ ❌ 금지  │        │
│  delete[] bufs                   │ ❌ 금지  │ ❌ 금지  │        │
│  new (ptr) GPUBuffer()           │ ❌ 금지  │ ✅ 허용  │ 풀링   │
│  new (ptr) GPUBuffer[10]         │ ❌ 금지  │ ✅ 허용  │ 풀링   │
│                                                                 │
│  스택 할당: GPUBuffer buf;       │ ✅ 허용  │ ✅ 허용  │ 권장   │
│  멤버 변수: GPUBuffer buf_;      │ ✅ 허용  │ ✅ 허용  │ 권장   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 비유: 가구 배달

```
┌─────────────────────────────────────────────────────────────────┐
│                    가구 배달 비유                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  일반 new (금지됨):                                             │
│  ─────────────────                                              │
│  "가구점에서 책상을 사서 집으로 배달해주세요"                   │
│  → 가구점이 직접 배달 트럭 부름 (힙 할당)                       │
│  → 책상 조립해서 배달 (생성자 호출)                             │
│                                                                 │
│  금지 이유: 배달된 책상을 반품할 때 문제                        │
│  (vtable 없어서 올바른 반품 처리 불가)                          │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Placement new (허용됨):                                        │
│  ──────────────────────                                         │
│  "제가 직접 트럭 가져왔어요, 여기에 책상만 조립해주세요"        │
│  → 트럭은 내가 가져옴 (메모리 이미 할당됨)                      │
│  → 가구점은 조립만 함 (생성자만 호출)                           │
│                                                                 │
│  허용 이유:                                                     │
│  - 트럭 관리는 내가 직접 함 (메모리 관리는 allocator가)         │
│  - 반품도 내가 알아서 처리 (소멸자 직접 호출)                   │
│  - 가구점 입장에서는 조립만 하면 됨                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Block Allocator와의 연계

### 커밋 #8 (Pooled shared_ptr) 이후 필요성

```cpp
// SharedBlockAllocatorImpl<T>::allocate() 내부
template<typename... ARG>
shared_ptr<T> allocate(ARG&&... args)
{
    // 1. 풀에서 메모리 가져오기
    RawStruct* raw = free_list.pop();

    // 2. Placement new로 객체 생성
    T* ptr = new (raw->data) T(std::forward<ARG>(args)...);
    //       ^^^^^^^^^^^^^^^^
    //       이게 가능하려면 placement new 허용 필요!

    // 3. shared_ptr 반환
    return make_ptr(ptr);
}
```

### 타임라인

```
커밋 #7 (11/20): 모든 new 금지 (placement new 포함)
         ↓
커밋 #8 (11/22): Pooled shared_ptr 도입 → placement new 필요!
         ↓
커밋 #11 (11/24): placement new 허용
```

커밋 #8 이후 Block Allocator가 Graphics 객체를 생성해야 했으나,
커밋 #7의 과도한 제한으로 컴파일 에러 발생.
커밋 #11에서 이를 해결.

---

## 영향받는 클래스

```cpp
// 세 클래스에 동일한 변경 적용

struct GPUBuffer final : public GPUResource {
    // ... placement new 허용
};

struct Texture final : public GPUResource {
    // ... placement new 허용
};

struct RaytracingAccelerationStructure final : public GPUResource {
    // ... placement new 허용
};
```

이 세 클래스가 `final`로 선언되어 있고 vtable이 없어서
동적 할당이 위험하지만, placement new는 메모리 관리를
외부(Block Allocator)에서 하므로 안전합니다.

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-28

### 적용 위치

| 파일 | 클래스 | 변경 내용 |
|------|--------|----------|
| GBackend.h | GPUBuffer | placement new 허용 |
| GBackend.h | Texture | placement new 허용 |
| GBackend.h | RaytracingAccelerationStructure | placement new 허용 |

### 코드 예시 (VizMotive)

```cpp
struct GPUBuffer final : public GPUResource
{
    // 일반 동적 할당 금지
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;

    // Placement new 허용
    static void* operator new(size_t, void* p) noexcept { return p; }
    static void* operator new[](size_t, void* p) noexcept { return p; }

    // ...
};
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | 커밋 #7에서 placement new도 함께 금지됨 |
| 증상 | Block Allocator에서 Graphics 객체 생성 불가 |
| 해결 | placement new만 허용 (메모리 할당 없이 생성자만 호출) |
| 대상 | GPUBuffer, Texture, RaytracingAccelerationStructure |
| VizMotive | ✅ 적용 완료 (2026-01-28) |

### 핵심 교훈

> **Placement New vs 일반 New의 차이**
>
> 일반 new: 메모리 할당 + 생성자 호출 (한 번에)
> Placement new: 생성자 호출만 (메모리는 이미 있음)
>
> vtable 없는 클래스에서 일반 new가 위험한 이유는
> delete 시 올바른 소멸자가 호출되지 않기 때문.
>
> Placement new는 메모리 관리를 외부에서 하므로
> 이 문제가 발생하지 않음.

> **Block Allocator 패턴**
>
> 1. 미리 메모리 블록 할당
> 2. 필요할 때 placement new로 객체 생성
> 3. 사용 후 소멸자 직접 호출 (`ptr->~T()`)
> 4. 메모리를 free_list로 반환 (실제 해제 안 함)
>
> 이 패턴을 위해 placement new가 필수적.
