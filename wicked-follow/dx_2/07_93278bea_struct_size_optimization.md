# 커밋 #7: 93278bea - Removed virtual tables from gfx objects, struct size reductions

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `93278bea` |
| 날짜 | 2025-11-20 |
| 작성자 | Turánszki János |
| 카테고리 | 최적화 / 메모리 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphics.h | 구조체 크기 최적화, vtable 제거, 타입 크기 축소 |
| wiGraphicsDevice_DX12.cpp/h | 관련 변경 적용 |

---

## 변경 1: BindFlag 타입 축소

### 변경 전

```cpp
enum class BindFlag {
    NONE = 0,
    VERTEX_BUFFER = 1 << 0,
    INDEX_BUFFER = 1 << 1,
    // ... (8개 플래그)
};
// 기본 크기: int (4바이트)
```

### 변경 후

```cpp
enum class BindFlag : uint8_t {
    NONE = 0,
    VERTEX_BUFFER = 1 << 0,
    INDEX_BUFFER = 1 << 1,
    // ... (8개 플래그)
};
// 크기: 1바이트
```

### 효과

```
┌─────────────────────────────────────────────────────────────────┐
│                    BindFlag 메모리 절약                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  변경 전: 4바이트 (int)                                         │
│  변경 후: 1바이트 (uint8_t)                                     │
│  절약: 3바이트/사용처                                           │
│                                                                 │
│  8개 플래그 = 8비트로 충분 (2^8 = 256 > 8)                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 변경 2: GPUBufferDesc 구조체 최적화

### 변경 전 (패딩 발생)

```cpp
struct GPUBufferDesc {
    uint64_t size = 0;           // 8바이트
    Usage usage = Usage::DEFAULT; // 4바이트
    // 4바이트 패딩 (8바이트 정렬)
    Format format = Format::UNKNOWN;
    uint32_t stride = 0;
    uint64_t alignment = 0;      // 8바이트 (uint64_t!)
    BindFlag bind_flags;         // 4바이트 (int)
    ResourceMiscFlag misc_flags;
};
// 총 크기: ~40바이트 이상 (패딩 포함)
```

### 변경 후 (패딩 최소화)

```cpp
struct GPUBufferDesc {
    uint64_t size = 0;           // 8바이트
    uint32_t stride = 0;         // 4바이트 (순서 변경)
    uint32_t alignment = 0;      // 4바이트 (uint32_t로 축소!)
    Usage usage = Usage::DEFAULT; // 4바이트
    Format format = Format::UNKNOWN; // 4바이트
    BindFlag bind_flags;         // 1바이트 (uint8_t)
    ResourceMiscFlag misc_flags; // 1바이트
    // 2바이트 패딩
};
// 총 크기: 28바이트
```

### 패딩 규칙

```
┌─────────────────────────────────────────────────────────────────┐
│                    구조체 패딩 규칙                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  규칙: 멤버는 자신의 정렬 크기의 배수 주소에 위치해야 함         │
│                                                                 │
│  예시 (변경 전):                                                │
│  offset 0:  uint64_t size        (8바이트, 정렬=8)              │
│  offset 8:  Usage usage          (4바이트, 정렬=4)              │
│  offset 12: [패딩 4바이트]       (다음이 uint64_t라서)          │
│  offset 16: uint64_t alignment   (8바이트, 정렬=8)              │
│  ...                                                            │
│                                                                 │
│  최적화 전략:                                                   │
│  1. 큰 타입(8바이트)을 먼저 배치                                │
│  2. 작은 타입(4바이트)을 다음에 배치                            │
│  3. 가장 작은 타입(1바이트)을 마지막에 배치                     │
│  → 패딩 최소화!                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 변경 3: GraphicsDeviceChild 기본 클래스 제거

### 변경 전

```cpp
// 기본 클래스 (vtable 포함)
struct GraphicsDeviceChild {
    std::shared_ptr<void> internal_state;
    virtual ~GraphicsDeviceChild() = default;  // vtable 포인터 8바이트!
};

// 파생 클래스들
struct Sampler : public GraphicsDeviceChild {
    SamplerDesc desc;
};

struct GPUBuffer : public GraphicsDeviceChild {
    GPUBufferDesc desc;
};
// ...
```

### 변경 후

```cpp
// 기본 클래스 없음, 각 구조체가 직접 멤버 보유
struct Sampler {
    vz::allocator::shared_ptr<void> internal_state;
    constexpr bool IsValid() const { return internal_state.IsValid(); }
    SamplerDesc desc;
};

struct GPUBuffer final : public GPUResource {
    GPUBufferDesc desc;
    constexpr bool IsValid() const { return internal_state.IsValid(); }
    // ...
};
```

### vtable 오버헤드

```
┌─────────────────────────────────────────────────────────────────┐
│                    Virtual Table (vtable) 오버헤드               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  virtual 소멸자가 있는 클래스:                                  │
│  ┌─────────────────────────────────┐                            │
│  │ vptr (vtable 포인터)  8바이트   │ ← 숨겨진 멤버!             │
│  │ internal_state        16바이트  │                            │
│  │ desc                  N바이트   │                            │
│  └─────────────────────────────────┘                            │
│                                                                 │
│  virtual 없는 클래스:                                           │
│  ┌─────────────────────────────────┐                            │
│  │ internal_state        16바이트  │                            │
│  │ desc                  N바이트   │                            │
│  └─────────────────────────────────┘                            │
│                                                                 │
│  절약: 8바이트/객체                                             │
│                                                                 │
│  1000개 GPUBuffer → 8KB 절약                                    │
│  10000개 Texture → 80KB 절약                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 변경 4: final 키워드 및 동적 할당 금지

### 변경 내용

```cpp
struct GPUBuffer final : public GPUResource {
    // final: 이 클래스를 상속 불가
    // → 컴파일러가 가상 함수 호출 최적화 가능 (devirtualization)

    // 동적 할당 금지
    static void* operator new(size_t) = delete;
    static void* operator new[](size_t) = delete;
    static void  operator delete(void*) = delete;
    static void  operator delete[](void*) = delete;

    // placement new는 허용 (BlockAllocator 등에서 필요)
    static void* operator new(size_t, void* p) noexcept { return p; }
    static void* operator new[](size_t, void* p) noexcept { return p; }
};
```

### 왜 동적 할당 금지?

```
┌─────────────────────────────────────────────────────────────────┐
│                    동적 할당 금지 이유                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. vtable이 없으므로 다형성(polymorphism) 불가                 │
│     - `delete base_ptr`로 파생 클래스 삭제 시 문제 발생         │
│     - 애초에 동적 할당을 막아서 실수 방지                       │
│                                                                 │
│  2. 스택/전역 할당 권장                                         │
│     - Graphics 객체는 보통 멤버 변수나 지역 변수로 사용         │
│     - 힙 할당 오버헤드 회피                                     │
│                                                                 │
│  3. placement new는 허용                                        │
│     - BlockAllocator에서 풀링된 메모리에 객체 생성 필요         │
│     - 수동 메모리 관리가 필요한 경우에만 허용                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 변경 5: GetMinOffsetAlignment 반환 타입 변경

### 변경 전

```cpp
virtual uint64_t GetMinOffsetAlignment(const GPUBufferDesc* desc) const = 0;
```

### 변경 후

```cpp
virtual uint32_t GetMinOffsetAlignment(const GPUBufferDesc* desc) const = 0;
```

### 이유

```
정렬 값은 보통 256, 512, 65536 등 작은 값
uint32_t (최대 4GB)로 충분
4바이트 절약 (반환값 + 일부 아키텍처에서 레지스터 효율)
```

---

## 전체 메모리 절약 요약

```
┌─────────────────────────────────────────────────────────────────┐
│                    구조체별 메모리 절약                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  구조체           │ 변경 전  │ 변경 후  │ 절약     │            │
│  ─────────────────┼──────────┼──────────┼──────────┤            │
│  BindFlag enum    │ 4바이트  │ 1바이트  │ 3바이트  │            │
│  GPUBufferDesc    │ ~40바이트│ 28바이트 │ ~12바이트│            │
│  각 Gfx 객체      │ +8바이트 │ +0바이트 │ 8바이트  │ (vtable)   │
│  GPUBufferDesc    │ 8바이트  │ 4바이트  │ 4바이트  │ (alignment)│
│                                                                 │
│  대규모 씬 예시 (10,000개 객체):                                │
│  - vtable 제거: ~80KB 절약                                      │
│  - 구조체 최적화: ~120KB 절약                                   │
│  - 총: ~200KB 절약                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-28

### 적용 위치

| 변경 항목 | 파일 | 라인 |
|-----------|------|------|
| `BindFlag : uint8_t` | GBackend.h | 382 |
| GPUBufferDesc 멤버 순서 | GBackend.h | 636-645 |
| GPUResource 분리 (no vtable) | GBackend.h | 844-865 |
| `GPUBuffer final` | GBackend.h | 867 |
| `Texture final` | GBackend.h | 882 |
| `operator delete = delete` | GBackend.h | 874-879 |
| placement new 허용 | GBackend.h | 878-879 |
| `IsValid() const` 각 구조체에 | GBackend.h | 829, 839, 847, ... |

---

## 요약

| 변경 항목 | 효과 |
|-----------|------|
| BindFlag : uint8_t | 3바이트/사용처 절약 |
| GPUBufferDesc 최적화 | ~12바이트 절약 (패딩 감소) |
| vtable 제거 | 8바이트/객체 절약 |
| alignment uint32_t | 4바이트 절약 |
| final 키워드 | devirtualization 최적화 가능 |
| operator delete 금지 | 잘못된 사용 방지 |

### 핵심 교훈

> **메모리 최적화 체크리스트**
>
> 1. **enum은 필요한 크기로**: 값이 256 이하면 `uint8_t`
> 2. **구조체 멤버 순서**: 큰 타입 먼저, 작은 타입 나중에
> 3. **불필요한 vtable 제거**: virtual이 정말 필요한지 검토
> 4. **타입 크기 검토**: uint64_t가 정말 필요한가?
> 5. **final 키워드**: 상속 안 할 클래스에 사용
