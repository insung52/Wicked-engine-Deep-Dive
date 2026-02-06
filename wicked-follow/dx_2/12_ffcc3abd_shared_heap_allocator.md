# 커밋 #12: ffcc3abd - Custom shared_ptr updates

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `ffcc3abd` |
| 날짜 | 2025-11-25 |
| 작성자 | Turánszki János |
| 카테고리 | 리팩토링 / 최적화 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiAllocator.h | 이름 변경 + SharedHeapAllocator 추가 |
| wiGraphicsDevice_DX12.cpp | AllocationHandler에 make_shared_single 사용 |

---

## 변경 1: 이름 변경 (WickedEngine)

### 변경 내용

```cpp
// 인터페이스 이름 변경
SharedBlockAllocator → SharedAllocator

// 구현체 이름 변경
SharedBlockAllocatorImpl → SharedBlockAllocator

// 전역 배열 이름 변경
block_allocators[] → shared_allocators[]
```

### 이름 변경 이유

```
┌─────────────────────────────────────────────────────────────────┐
│              이름 변경 이유                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  변경 전 (커밋 #8):                                             │
│  ─────────────────                                              │
│  SharedBlockAllocator (인터페이스)                              │
│       └── SharedBlockAllocatorImpl<T> (구현체)                  │
│                                                                 │
│  문제: Block Allocator만 있었음                                 │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  변경 후 (커밋 #12):                                            │
│  ─────────────────                                              │
│  SharedAllocator (인터페이스) ← 더 일반적인 이름                │
│       ├── SharedBlockAllocator<T> (블록 기반 풀링)              │
│       └── SharedHeapAllocator<T> (개별 힙 할당) ← 새로 추가!    │
│                                                                 │
│  이제 두 가지 할당 전략을 지원하므로                            │
│  인터페이스 이름을 더 일반적으로 변경                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### VizMotive 적용 여부

**스킵**: VizMotive는 DLL 경계 문제로 다른 구조를 사용하므로 이름 변경 불필요.

---

## 변경 2: SharedHeapAllocator 추가

### 왜 필요한가?

```
┌─────────────────────────────────────────────────────────────────┐
│              Block Allocator vs Heap Allocator                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SharedBlockAllocator (기존):                                   │
│  ─────────────────────────────                                  │
│  - 같은 타입 객체 256개를 한 블록에 풀링                        │
│  - 빈번하게 생성/삭제되는 객체에 최적                           │
│  - 예: Texture_DX12, GPUBuffer_DX12, Sampler_DX12               │
│                                                                 │
│  문제: 한 번만 생성되는 객체에는 비효율적                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ AllocationHandler 예시:                                  │   │
│  │                                                          │   │
│  │ GraphicsDevice당 1개만 존재                              │   │
│  │ 256개 블록 할당 → 255개 낭비!                            │   │
│  │                                                          │   │
│  │ Block:                                                   │   │
│  │ ┌────┬────┬────┬────┬────┬─────────────────────────────┐│   │
│  │ │ 사용│    │    │    │    │  ... 255개 빈 슬롯 ...     ││   │
│  │ └────┴────┴────┴────┴────┴─────────────────────────────┘│   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  SharedHeapAllocator (새로 추가):                               │
│  ─────────────────────────────────                              │
│  - 개별 힙 할당 (new/delete)                                    │
│  - 한 번만 생성되거나 드물게 사용되는 객체에 적합               │
│  - 풀링 오버헤드 없음                                           │
│  - 예: AllocationHandler                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### SharedHeapAllocator 구현

```cpp
template<typename T>
struct SharedHeapAllocator final : public SharedAllocator
{
    const uint8_t allocator_id = register_shared_allocator(this);

    // 개별 힙 할당 (블록 없음)
    struct RawStruct
    {
        uint8_t data[sizeof(T)];
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };

    template<typename... ARG>
    shared_ptr<T> allocate(ARG&&... args)
    {
        // 개별 new 호출 (풀링 없음)
        RawStruct* raw = new RawStruct();

        T* ptr = new (raw->data) T(std::forward<ARG>(args)...);
        init_refcount(ptr);

        shared_ptr<T> result;
        // ... handle 설정 ...
        return result;
    }

    // 메모리 해제 (weak_refcount가 0이 될 때)
    void reclaim(void* ptr)
    {
        RawStruct* raw = reinterpret_cast<RawStruct*>(ptr);
        delete raw;  // 개별 delete
    }

    // refcount 관리는 SharedBlockAllocator와 동일
    // ...
};
```

### 비교: Block vs Heap

```
┌─────────────────────────────────────────────────────────────────┐
│              SharedBlockAllocator vs SharedHeapAllocator         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                        │ Block          │ Heap                  │
│  ──────────────────────┼────────────────┼─────────────────────  │
│  할당 방식             │ 풀에서 가져옴  │ new 호출              │
│  해제 방식             │ 풀로 반환      │ delete 호출           │
│  메모리 레이아웃       │ 연속 (배열)    │ 분산 (개별)           │
│  할당 속도             │ O(1) 빠름      │ O(?) 느림             │
│  메모리 효율 (빈번)    │ 좋음           │ 나쁨 (단편화)         │
│  메모리 효율 (드문)    │ 나쁨 (낭비)    │ 좋음 (필요한 만큼)    │
│  ──────────────────────┼────────────────┼─────────────────────  │
│  적합한 사용처         │ Texture_DX12   │ AllocationHandler     │
│                        │ GPUBuffer_DX12 │ (디바이스당 1개)      │
│                        │ Sampler_DX12   │                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 변경 3: make_shared_single 함수 추가

### 함수 정의

```cpp
// Block allocator 기반 (기존) - 빈번한 객체용
template<typename T, size_t block_size = 256, typename... ARG>
inline shared_ptr<T> make_shared(ARG&&... args)
{
    return shared_block_allocator<T, block_size>->allocate(std::forward<ARG>(args)...);
}

// Heap allocator 기반 (새로 추가) - 단일 객체용
template<typename T, typename... ARG>
inline shared_ptr<T> make_shared_single(ARG&&... args)
{
    return shared_heap_allocator<T>->allocate(std::forward<ARG>(args)...);
}
```

### 사용 예시

```cpp
// 빈번하게 생성되는 객체 → make_shared (블록 풀링)
auto texture = make_shared<Texture_DX12>();
auto buffer = make_shared<GPUBuffer_DX12>();

// 한 번만 생성되는 객체 → make_shared_single (힙 할당)
auto handler = make_shared_single<AllocationHandler>();
```

---

## DX12 변경: AllocationHandler

### 변경 내용

```cpp
// 변경 전: Block allocator 사용 (비효율적)
allocationhandler = make_shared<AllocationHandler>();
// → 256개 슬롯 할당, 1개만 사용

// 변경 후: Heap allocator 사용 (효율적)
allocationhandler = make_shared_single<AllocationHandler>();
// → 1개만 할당
```

### AllocationHandler란?

```
┌─────────────────────────────────────────────────────────────────┐
│              AllocationHandler 역할                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  GraphicsDevice_DX12 내부:                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ allocationhandler = make_shared_single<AllocationHandler>│   │
│  │                                                          │   │
│  │ AllocationHandler:                                       │   │
│  │ ┌────────────────────────────────────────────────────┐   │   │
│  │ │ - D3D12MA::Allocator (GPU 메모리 관리)             │   │   │
│  │ │ - Descriptor heap 관리                             │   │   │
│  │ │ - 다양한 descriptor allocator들                    │   │   │
│  │ │ - Deferred destruction queue                       │   │   │
│  │ └────────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  특징:                                                          │
│  - GraphicsDevice당 1개만 존재                                  │
│  - 크기가 큼 (많은 내부 상태)                                   │
│  - 앱 수명 동안 유지                                            │
│                                                                 │
│  → Block allocator 부적합, Heap allocator 적합                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 비유: 주차장 vs 길거리 주차

```
┌─────────────────────────────────────────────────────────────────┐
│                    주차 비유                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SharedBlockAllocator (주차장):                                 │
│  ─────────────────────────────                                  │
│  - 256칸짜리 주차장을 미리 확보                                 │
│  - 차가 들어오면 빈 칸에 주차                                   │
│  - 차가 나가면 칸 비워둠 (다음 차를 위해)                       │
│                                                                 │
│  좋은 경우: 많은 차가 자주 드나드는 쇼핑몰                      │
│  나쁜 경우: 한 대만 주차하는 개인 집                            │
│            → 256칸 중 255칸 낭비!                               │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  SharedHeapAllocator (길거리 주차):                             │
│  ─────────────────────────────────                              │
│  - 필요할 때 주차 공간 찾아서 주차                              │
│  - 나갈 때 그 공간 반납                                         │
│                                                                 │
│  좋은 경우: 가끔 한 대만 주차하는 경우                          │
│  나쁜 경우: 많은 차가 자주 드나드는 경우                        │
│            → 매번 주차 공간 찾아야 함 (느림)                    │
│                                                                 │
│  AllocationHandler = 사장님 차 (항상 한 대, 오래 주차)          │
│  → 길거리 주차가 더 효율적!                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## VizMotive 적용 현황

### 부분 적용 ✅

**적용 일자**: 2026-01-28

### 적용한 부분

| 파일 | 변경 내용 |
|------|----------|
| Allocator.h | `SharedHeapAllocator<T>` 템플릿 추가 |
| Allocator.h | `make_shared_single<T>()` 함수 추가 |
| GraphicsDevice_DX12.h | `allocationhandler` 타입 변경 |
| GraphicsDevice_DX12.cpp | `make_shared_single<AllocationHandler>()` 사용 |
| GraphicsDevice_DX12.cpp | 8개 internal struct의 `allocationhandler` 타입 변경 |

### 스킵한 부분

| 변경 | 스킵 이유 |
|------|----------|
| SharedBlockAllocator → SharedAllocator 이름 변경 | VizMotive는 DLL 경계 대응으로 다른 구조 사용 |
| block_allocators → shared_allocators 이름 변경 | 위와 동일 |

### VizMotive SharedHeapAllocator 구현

```cpp
// Allocator.h
template<typename T>
struct SharedHeapAllocator final : public SharedBlockAllocator
{
    struct RawStruct
    {
        uint8_t data[sizeof(T)];
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };

    template<typename... ARG>
    shared_ptr<T> allocate(ARG&&... args)
    {
        RawStruct* raw = new RawStruct();
        T* ptr = new (raw->data) T(std::forward<ARG>(args)...);
        init_refcount(ptr);

        shared_ptr<T> result;
        result.ptr = ptr;
        result.allocator = this;  // VizMotive: 포인터 직접 저장
        return result;
    }

    void reclaim(void* ptr)
    {
        RawStruct* raw = static_cast<RawStruct*>(ptr);
        delete raw;
    }

    // refcount 관리 함수들...
};

// 헬퍼 함수
template<typename T, typename... ARG>
inline shared_ptr<T> make_shared_single(ARG&&... args)
{
    static SharedHeapAllocator<T> allocator;
    return allocator.allocate(std::forward<ARG>(args)...);
}
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 변경 1 | 이름 변경: SharedBlockAllocator → SharedAllocator (VizMotive 스킵) |
| 변경 2 | SharedHeapAllocator 추가 - 단일 객체용 힙 할당 |
| 변경 3 | make_shared_single() 헬퍼 함수 추가 |
| 적용 대상 | AllocationHandler (디바이스당 1개) |
| VizMotive | ✅ 부분 적용 (HeapAllocator + make_shared_single) |

### 핵심 교훈

> **객체 생성 패턴에 맞는 할당자 선택**
>
> - 빈번하게 생성/삭제: Block Allocator (풀링)
> - 한 번만 생성/오래 유지: Heap Allocator (개별 할당)
>
> 모든 것에 Block Allocator를 쓰면 메모리 낭비 발생.
> 객체의 수명 패턴을 고려하여 적절한 할당자 선택.

> **Two-tier 할당 전략**
>
> ```cpp
> // 빈번한 객체: 풀링
> auto tex = make_shared<Texture_DX12>();
>
> // 단일 객체: 힙
> auto handler = make_shared_single<AllocationHandler>();
> ```
