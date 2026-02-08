# Topic: Struct Optimization (구조체 최적화)

## 개요

Graphics 객체 구조체의 메모리 사용량 최적화.

## 관련 커밋

| 순서 | 커밋 | 날짜 | 핵심 변경 |
|------|------|------|----------|
| 1 | [dx2 #7 `93278bea`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/07_93278bea_struct_size_optimization.md) | 2025-11-20 | vtable 제거, 구조체 크기 축소 |
| 2 | [dx2 #14 `973c2850`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/14_973c2850_internal_struct_final.md) | 2025-11-26 | internal structures final 키워드 |

---

## 변화 흐름

### 1. 구조체 크기 최적화 (dx2 #7)

**변경 1: BindFlag enum 크기 축소**

```cpp
// 변경 전: 4바이트
enum BindFlag { ... };

// 변경 후: 1바이트
enum BindFlag : uint8_t { ... };
```

**변경 2: GPUBufferDesc 멤버 재배치**

```cpp
// 변경 전: 패딩으로 인한 낭비
struct GPUBufferDesc {
    uint64_t size;         // 8 bytes, offset 0
    BindFlag bind_flags;   // 4 bytes, offset 8
    // 4 bytes padding
    uint64_t alignment;    // 8 bytes, offset 16
    // ...
};

// 변경 후: 패딩 최소화
struct GPUBufferDesc {
    uint64_t size;         // 8 bytes
    uint32_t alignment;    // 4 bytes (uint64_t → uint32_t)
    BindFlag bind_flags;   // 1 byte (uint8_t)
    // ...
};
```

**변경 3: GraphicsDeviceChild 제거 (vtable 제거)**

```cpp
// 변경 전
struct GraphicsDeviceChild {
    virtual ~GraphicsDeviceChild() = default;  // vtable 8바이트
};

struct GPUBuffer : public GraphicsDeviceChild { ... };

// 변경 후
struct GPUBuffer final {  // 상속 없음, final로 표시
    // vtable 없음!
};
```

**메모리 절약**:
- vtable 포인터: 객체당 8바이트 절약
- enum 크기: 플래그당 3바이트 절약
- 패딩: 구조체당 수 바이트 절약

**변경 4: operator delete 금지**

```cpp
struct GPUBuffer final {
    void operator delete(void*) = delete;  // 힙 할당 금지
};

// 의도: 스택/정적 할당 또는 placement new만 허용
// Block Allocator와 함께 사용
```

---

### 2. Internal Structures final 키워드 (dx2 #14)

**대상**: `Texture_DX12`, `BVH_DX12`

```cpp
// 변경 전
struct Texture_DX12 : public Resource_DX12 { ... };

// 변경 후
struct Texture_DX12 final : public Resource_DX12 { ... };
```

**효과: Devirtualization**

```
┌─────────────────────────────────────────────────────────────────┐
│  Devirtualization                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  final이 없으면:                                                │
│  - 컴파일러가 파생 클래스 존재 가능성 고려                      │
│  - 가상 함수 호출은 vtable 통해야 함                            │
│  - 인라이닝 불가                                                │
│                                                                 │
│  final이 있으면:                                                │
│  - 이 클래스가 최종임을 컴파일러가 앎                           │
│  - 가상 함수를 직접 호출로 변환 가능                            │
│  - 인라이닝 가능 → 성능 향상                                    │
│                                                                 │
│  obj->virtualMethod();                                          │
│                                                                 │
│  [final 없음]                [final 있음]                       │
│  load vtable pointer         직접 호출 (또는 인라인)            │
│  load function pointer       → 더 빠름                          │
│  indirect call                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 핵심 개념

### 구조체 패딩

```
┌─────────────────────────────────────────────────────────────────┐
│  구조체 패딩 예시                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  struct Bad {                    struct Good {                  │
│      uint8_t a;   // 1 byte          uint64_t c;  // 8 bytes   │
│      // 7 bytes padding              uint32_t b;  // 4 bytes   │
│      uint64_t c;  // 8 bytes         uint8_t a;   // 1 byte    │
│      uint32_t b;  // 4 bytes         // 3 bytes padding        │
│      // 4 bytes padding          };                             │
│  };                                                             │
│                                                                 │
│  sizeof(Bad) = 24              sizeof(Good) = 16               │
│                                                                 │
│  규칙: 큰 타입부터 작은 타입 순서로 배치                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### vtable 오버헤드

```
┌─────────────────────────────────────────────────────────────────┐
│  vtable 구조                                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  class Base {                                                   │
│      virtual ~Base() = default;                                 │
│  };                                                             │
│                                                                 │
│  객체 메모리 레이아웃:                                          │
│  ┌────────────────────────────────────────────┐                 │
│  │ vptr (8 bytes) │ ... 멤버 변수들 ...       │                 │
│  └────────────────────────────────────────────┘                 │
│        ↓                                                        │
│  ┌────────────────────────────────────────────┐                 │
│  │ vtable                                     │                 │
│  │ ├── &Base::~Base()                         │                 │
│  │ └── (다른 가상 함수들...)                  │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
│  오버헤드: 객체당 8바이트 (64비트 시스템)                       │
│  10,000개 객체 → 80KB 낭비                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 교훈

### 1. 가상 함수 필요성 재검토
상속이 필요 없는 Graphics 객체는 가상 함수 제거.
객체당 8바이트 절약 + devirtualization 성능 향상.

### 2. 멤버 순서 최적화
큰 타입부터 작은 타입 순서로 배치하면 패딩 최소화.
특히 빈번히 생성되는 객체일수록 효과 큼.

### 3. enum 크기 명시
`enum BindFlag : uint8_t`처럼 기저 타입 명시.
기본값(int, 4바이트)보다 작은 타입 사용 가능.

### 4. final 키워드
내부 구현 클래스에 `final` 추가하면 컴파일러 최적화 기회 증가.

---

## VizMotive 적용

모든 커밋 적용 완료:
- `GBackend.h`: BindFlag uint8_t, GPUBufferDesc/TextureDesc 재배치
- `GBackend.h`: GraphicsDeviceChild 제거, final 키워드 추가
- `GraphicsDevice_DX12.cpp`: Texture_DX12, BVH_DX12에 final 추가
