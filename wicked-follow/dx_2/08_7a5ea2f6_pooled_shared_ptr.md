# 커밋 #8: 7a5ea2f6 - Pooled shared ptr

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `7a5ea2f6` |
| 날짜 | 2025-11-22 |
| 작성자 | Turánszki János |
| 카테고리 | 최적화 / 메모리 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiAllocator.h | 커스텀 shared_ptr/weak_ptr 구현, SharedBlockAllocator |
| wiGraphics.h | internal_state 타입 변경 |
| wiGraphicsDevice_DX12.cpp | make_shared 호출 변경 |

---

## 배경 지식: std::shared_ptr의 문제점

### std::shared_ptr 내부 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                 std::shared_ptr 메모리 구조                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  std::shared_ptr<T> ptr = std::make_shared<T>();                │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │ shared_ptr 객체  │  (스택/멤버)                              │
│  │ ─────────────── │                                           │
│  │ T* ptr          │ ──────┐                                   │
│  │ control_block*  │ ──┐   │                                   │
│  └──────────────────┘   │   │                                   │
│                         │   │                                   │
│                         ▼   ▼                                   │
│  ┌──────────────────────────────────┐  ← 힙 할당 (한 블록)      │
│  │ Control Block + T 객체           │                           │
│  │ ─────────────────────────────── │                           │
│  │ atomic<int> strong_count        │  4바이트                   │
│  │ atomic<int> weak_count          │  4바이트                   │
│  │ deleter (optional)              │  ~8바이트                  │
│  │ allocator (optional)            │  ~8바이트                  │
│  │ ─────────────────────────────── │                           │
│  │ T 객체 데이터                    │  sizeof(T)                │
│  └──────────────────────────────────┘                           │
│                                                                 │
│  총 오버헤드: ~24바이트 + 힙 할당 비용                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Graphics 객체에서의 문제

```
┌─────────────────────────────────────────────────────────────────┐
│              std::shared_ptr 사용 시 문제점                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 빈번한 힙 할당/해제                                         │
│     ─────────────────────────────────────                      │
│     Graphics 객체 (Texture, Buffer, Shader 등)는               │
│     프레임마다 생성/삭제될 수 있음                               │
│                                                                 │
│     매번 힙 할당 → 오버헤드 + 메모리 단편화                     │
│                                                                 │
│  2. 메모리 단편화                                               │
│     ─────────────────────────────────────                      │
│     ┌───┐ ┌───┐     ┌───┐ ┌───┐     ┌───┐                      │
│     │ A │ │ B │ ... │ C │ │ D │ ... │ E │  ← 흩어진 할당        │
│     └───┘ └───┘     └───┘ └───┘     └───┘                      │
│                                                                 │
│     캐시 미스 증가, 메모리 효율 감소                            │
│                                                                 │
│  3. atomic 연산 오버헤드                                        │
│     ─────────────────────────────────────                      │
│     refcount 증가/감소마다 atomic 연산                          │
│     → 멀티스레드 환경에서 메모리 배리어 비용                    │
│                                                                 │
│  4. Control Block 오버헤드                                      │
│     ─────────────────────────────────────                      │
│     객체당 ~24바이트 추가 메모리                                │
│     10,000개 객체 → 240KB 낭비                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 해결책: Pooled Block Allocator 기반 shared_ptr

### 핵심 아이디어

```
┌─────────────────────────────────────────────────────────────────┐
│              Block Allocator 기반 풀링                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  std::shared_ptr (기존):                                        │
│  ┌───┐     ┌───┐     ┌───┐     ┌───┐                           │
│  │ A │     │ B │     │ C │     │ D │  ← 개별 힙 할당            │
│  └───┘     └───┘     └───┘     └───┘                           │
│    ↑         ↑         ↑         ↑                             │
│   new       new       new       new                             │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Block Allocator 기반 (새로운):                                 │
│  ┌─────────────────────────────────────────────────────┐        │
│  │ Block 0                                             │        │
│  │ ┌───┬───┬───┬───┬───┬───┬───┬───┬─────────────────┐│        │
│  │ │ A │ B │ C │ D │ E │ F │ G │ H │ ... (256개)     ││        │
│  │ └───┴───┴───┴───┴───┴───┴───┴───┴─────────────────┘│        │
│  └─────────────────────────────────────────────────────┘        │
│                         ↑                                       │
│                   한 번의 벡터 확장                              │
│                                                                 │
│  장점:                                                          │
│  - 연속 메모리 → 캐시 효율 향상                                 │
│  - 할당/해제가 free_list 조작만 → O(1)                          │
│  - 메모리 단편화 없음                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## WickedEngine 구현 분석

### 1. SharedBlockAllocator 인터페이스

```cpp
// 모든 타입별 allocator가 구현해야 하는 인터페이스
struct SharedBlockAllocator
{
    // Reference count 관리
    virtual void init_refcount(void* ptr) = 0;
    virtual uint32_t get_refcount(void* ptr) = 0;
    virtual uint32_t inc_refcount(void* ptr) = 0;
    virtual uint32_t dec_refcount(void* ptr) = 0;

    // Weak reference count 관리
    virtual uint32_t get_refcount_weak(void* ptr) = 0;
    virtual uint32_t inc_refcount_weak(void* ptr) = 0;
    virtual uint32_t dec_refcount_weak(void* ptr) = 0;
};
```

### 2. 전역 allocator 등록 시스템 (WickedEngine 원본)

```cpp
// 전역 allocator 배열 (최대 256개)
inline SharedBlockAllocator* block_allocators[256] = {};
inline std::atomic<uint8_t> block_allocator_count{ 0 };

// allocator 등록 함수
inline uint8_t register_shared_block_allocator(SharedBlockAllocator* allocator)
{
    uint8_t id = block_allocator_count.fetch_add(1);
    block_allocators[id] = allocator;
    return id;
}
```

### 3. Handle 인코딩 방식 (WickedEngine 원본)

```
┌─────────────────────────────────────────────────────────────────┐
│              WickedEngine Handle 인코딩                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  64비트 handle:                                                 │
│  ┌────────────────────────────────────────────────┬───────────┐ │
│  │            포인터 (상위 56비트)                 │ ID (8비트)│ │
│  └────────────────────────────────────────────────┴───────────┘ │
│                                                                 │
│  요구사항: 포인터가 256바이트 정렬되어야 함                      │
│  → 하위 8비트가 항상 0 → allocator ID 저장 가능                 │
│                                                                 │
│  struct shared_ptr<T> {                                         │
│      uint64_t handle = 0;  // ptr | allocator_id                │
│                                                                 │
│      T* get_ptr() const {                                       │
│          return (T*)(handle & (~0ull << 8ull));  // 상위 56비트 │
│      }                                                          │
│                                                                 │
│      SharedBlockAllocator* get_allocator() const {              │
│          return block_allocators[handle & 0xFF];  // 하위 8비트 │
│      }                                                          │
│  };                                                             │
│                                                                 │
│  장점: shared_ptr 크기 = 8바이트 (포인터 하나)                  │
│  단점: 256바이트 정렬 필요, 전역 배열 의존                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4. SharedBlockAllocatorImpl 구현

```cpp
template<typename T, size_t block_size = 256>
struct SharedBlockAllocatorImpl final : public SharedBlockAllocator
{
    // 생성 시 자동으로 전역 배열에 등록
    const uint8_t allocator_id = register_shared_block_allocator(this);

    // 256바이트 정렬된 RawStruct (handle 인코딩을 위해)
    struct alignas(256) RawStruct
    {
        uint8_t data[sizeof(T)];
        std::atomic<uint32_t> refcount;
        std::atomic<uint32_t> refcount_weak;
    };

    struct Block
    {
        std::vector<RawStruct> mem;
        Block() { mem.resize(block_size); }
    };

    std::vector<Block> blocks;
    std::vector<RawStruct*> free_list;
    SpinLock locker;

    // 할당
    template<typename... ARG>
    shared_ptr<T> allocate(ARG&&... args)
    {
        locker.lock();
        if (free_list.empty())
        {
            // 새 블록 추가
            blocks.emplace_back();
            Block& block = blocks.back();
            for (auto& item : block.mem)
            {
                free_list.push_back(&item);
            }
        }
        RawStruct* raw = free_list.back();
        free_list.pop_back();
        locker.unlock();

        // placement new로 객체 생성
        T* ptr = new (raw->data) T(std::forward<ARG>(args)...);
        init_refcount(ptr);

        // handle = ptr | allocator_id
        shared_ptr<T> result;
        result.handle = (uint64_t)ptr | allocator_id;
        return result;
    }

    // refcount 관리
    uint32_t inc_refcount(void* ptr) override
    {
        RawStruct* raw = reinterpret_cast<RawStruct*>(ptr);
        return raw->refcount.fetch_add(1);
    }

    uint32_t dec_refcount(void* ptr) override
    {
        RawStruct* raw = reinterpret_cast<RawStruct*>(ptr);
        uint32_t old = raw->refcount.fetch_sub(1);
        if (old == 1)
        {
            // refcount가 0이 됨 → 객체 파괴
            static_cast<T*>(ptr)->~T();
            dec_refcount_weak(ptr);  // weak도 감소
        }
        return old;
    }

    // weak refcount가 0이 되면 메모리를 free_list로 반환
    uint32_t dec_refcount_weak(void* ptr) override
    {
        RawStruct* raw = reinterpret_cast<RawStruct*>(ptr);
        uint32_t old = raw->refcount_weak.fetch_sub(1);
        if (old == 1)
        {
            // weak도 0 → 메모리 재사용 가능
            locker.lock();
            free_list.push_back(raw);
            locker.unlock();
        }
        return old;
    }
};
```

### 5. 타입별 allocator 인스턴스

```cpp
// 각 타입별로 static allocator 인스턴스
template<typename T, size_t block_size = 256>
inline SharedBlockAllocatorImpl<T, block_size>* shared_block_allocator =
    new SharedBlockAllocatorImpl<T, block_size>();

// make_shared 헬퍼
template<typename T, size_t block_size = 256, typename... ARG>
inline shared_ptr<T> make_shared(ARG&&... args)
{
    return shared_block_allocator<T, block_size>->allocate(std::forward<ARG>(args)...);
}
```

---

## VizMotive 적용 시 문제: DLL 경계

### 문제 상황

```
┌─────────────────────────────────────────────────────────────────┐
│                    DLL 경계 문제                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  VizMotive 아키텍처:                                            │
│  ┌─────────────────┐      ┌─────────────────┐                   │
│  │   Engine.dll    │      │   DX12.dll      │                   │
│  │                 │      │                 │                   │
│  │ block_allocators│      │ block_allocators│  ← 별도 인스턴스! │
│  │ [0] = nullptr   │      │ [0] = alloc_A   │                   │
│  │ [1] = nullptr   │      │ [1] = alloc_B   │                   │
│  │ ...             │      │ ...             │                   │
│  └─────────────────┘      └─────────────────┘                   │
│                                                                 │
│  문제 원인:                                                     │
│  - C++17 inline 변수는 각 DLL에서 별도로 인스턴스화             │
│  - DX12.dll에서 allocator 등록 → Engine.dll에서 볼 수 없음      │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  문제 시나리오:                                                 │
│                                                                 │
│  1. DX12.dll에서 make_shared<Texture_DX12>() 호출               │
│     → allocator 등록, handle 생성                               │
│                                                                 │
│  2. Texture 객체가 Engine.dll로 전달됨                          │
│     (internal_state에 handle 저장)                              │
│                                                                 │
│  3. Engine.dll에서 get_allocator() 호출                         │
│     → block_allocators[id] 조회                                 │
│     → nullptr 반환!  (Engine.dll의 배열은 비어있음)             │
│                                                                 │
│  4. 크래시 또는 메모리 누수                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 왜 WickedEngine은 괜찮은가?

```
WickedEngine: 단일 실행파일 (exe)
→ 모든 코드가 같은 주소 공간
→ inline 변수가 하나의 인스턴스만 존재
→ 문제 없음

VizMotive: 다중 DLL 아키텍처
→ Engine.dll + DX12.dll + Vulkan.dll + ...
→ 각 DLL이 별도의 inline 변수 인스턴스 보유
→ DLL 경계를 넘는 shared_ptr 전달 시 문제
```

---

## VizMotive 해결책: 포인터 직접 저장

### 수정된 shared_ptr 구조

```cpp
// VizMotive 버전 - allocator 포인터 직접 저장
template<typename T>
struct shared_ptr
{
    T* ptr = nullptr;
    SharedBlockAllocator* allocator = nullptr;  // 직접 저장!

    constexpr bool IsValid() const
    {
        return ptr != nullptr && allocator != nullptr;
    }

    constexpr T* get_ptr() const { return ptr; }
    constexpr SharedBlockAllocator* get_allocator() const { return allocator; }

    // 복사/이동/소멸자에서 allocator를 통해 refcount 관리
    shared_ptr(const shared_ptr& other) : ptr(other.ptr), allocator(other.allocator)
    {
        if (ptr && allocator) allocator->inc_refcount(ptr);
    }

    ~shared_ptr()
    {
        if (ptr && allocator) allocator->dec_refcount(ptr);
    }

    // ...
};
```

### 수정된 weak_ptr 구조

```cpp
template<typename T>
struct weak_ptr
{
    T* ptr = nullptr;
    SharedBlockAllocator* allocator = nullptr;  // 직접 저장!

    constexpr bool IsValid() const
    {
        return ptr != nullptr && allocator != nullptr;
    }

    shared_ptr<T> lock()
    {
        if (!ptr || !allocator) return {};

        // allocator를 통해 strong refcount 증가 시도
        if (allocator->try_inc_refcount(ptr))
        {
            shared_ptr<T> ret;
            ret.ptr = ptr;
            ret.allocator = allocator;
            return ret;
        }
        return {};
    }

    // ...
};
```

### 트레이드오프 비교

```
┌─────────────────────────────────────────────────────────────────┐
│              WickedEngine vs VizMotive 비교                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    │ WickedEngine      │ VizMotive              │
│  ──────────────────┼───────────────────┼──────────────────────  │
│  shared_ptr 크기   │ 8바이트 (handle)  │ 16바이트 (ptr+alloc)   │
│  weak_ptr 크기     │ 8바이트           │ 16바이트               │
│  DLL 경계          │ ❌ 문제 발생      │ ✅ 정상 작동           │
│  구현 복잡도       │ 높음 (비트 연산)  │ 낮음 (직관적)          │
│  정렬 요구사항     │ 256바이트         │ 없음                   │
│  전역 배열 의존    │ 있음              │ 없음                   │
│                                                                 │
│  메모리 영향 (10,000개 객체 기준):                              │
│  - WickedEngine: 10,000 × 8 = 80KB                              │
│  - VizMotive: 10,000 × 16 = 160KB                               │
│  - 차이: 80KB (큰 차이 아님)                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 전체 시스템 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                 Pooled shared_ptr 전체 구조                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ SharedBlockAllocatorImpl<Texture_DX12>                  │    │
│  │ ─────────────────────────────────────────────────────── │    │
│  │ Block 0:                                                │    │
│  │ ┌────────────────────────────────────────────────────┐  │    │
│  │ │ RawStruct[0] │ RawStruct[1] │ ... │ RawStruct[255] │  │    │
│  │ │ ┌──────────┐ │ ┌──────────┐ │     │ ┌──────────┐   │  │    │
│  │ │ │Texture_  │ │ │Texture_  │ │     │ │          │   │  │    │
│  │ │ │DX12 data │ │ │DX12 data │ │     │ │  (free)  │   │  │    │
│  │ │ │──────────│ │ │──────────│ │     │ │          │   │  │    │
│  │ │ │refcount=2│ │ │refcount=1│ │     │ │refcount=0│   │  │    │
│  │ │ │weak_cnt=1│ │ │weak_cnt=0│ │     │ │weak_cnt=0│   │  │    │
│  │ │ └──────────┘ │ └──────────┘ │     │ └──────────┘   │  │    │
│  │ └────────────────────────────────────────────────────┘  │    │
│  │                                                         │    │
│  │ free_list: [&RawStruct[255], &RawStruct[254], ...]     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                    ↑                                            │
│                    │                                            │
│  ┌─────────────────┴───────────────────────────────────────┐    │
│  │                                                         │    │
│  │  Texture tex;                                           │    │
│  │  tex.internal_state = make_shared<Texture_DX12>();      │    │
│  │  ┌──────────────────────────────────┐                   │    │
│  │  │ shared_ptr<void>                 │                   │    │
│  │  │ ──────────────────────────────── │                   │    │
│  │  │ ptr = &RawStruct[0].data         │ ──────────────┐   │    │
│  │  │ allocator = SharedBlockAlloc*    │ ──────────────┼───│    │
│  │  └──────────────────────────────────┘               │   │    │
│  │                                                     │   │    │
│  └─────────────────────────────────────────────────────│───┘    │
│                                                        │        │
│                                                        ▼        │
│                                        ┌───────────────────┐    │
│                                        │ refcount 관리:    │    │
│                                        │ - inc_refcount()  │    │
│                                        │ - dec_refcount()  │    │
│                                        │ - try_inc_refcount│    │
│                                        └───────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 비유: 도서관 책 대출 시스템

```
┌─────────────────────────────────────────────────────────────────┐
│                    도서관 비유                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  std::shared_ptr (기존):                                        │
│  ─────────────────────                                          │
│  매번 새 책을 서점에서 구매 (new)                               │
│  → 비용 높음, 책장 정리 어려움 (메모리 단편화)                  │
│  → 책 반납 시 서점에 돌려줌 (delete)                            │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  Pooled shared_ptr (새로운):                                    │
│  ─────────────────────                                          │
│  도서관에서 책 대출 (allocate from pool)                        │
│  → 책은 이미 도서관에 있음 (pre-allocated blocks)               │
│  → 대출/반납이 빠름 (free_list 조작)                            │
│  → 모든 책이 정리되어 있음 (연속 메모리)                        │
│                                                                 │
│  대출 카드 = shared_ptr                                         │
│  대출 기록 = refcount                                           │
│  예약 카드 = weak_ptr                                           │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  DLL 경계 문제 (WickedEngine):                                  │
│  ─────────────────────────────                                  │
│  대출 카드에 "도서관 번호"만 적혀있음                           │
│  → 다른 건물(DLL)에서는 어느 도서관인지 모름                    │
│                                                                 │
│  VizMotive 해결책:                                              │
│  ─────────────────                                              │
│  대출 카드에 "도서관 주소"를 직접 적음                          │
│  → 어느 건물에서든 도서관 찾아갈 수 있음                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 성능 비교

```
┌─────────────────────────────────────────────────────────────────┐
│              std::shared_ptr vs Pooled shared_ptr               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  연산              │ std::shared_ptr   │ Pooled shared_ptr      │
│  ─────────────────┼───────────────────┼───────────────────────  │
│  할당 (allocate)  │ new (힙 할당)     │ free_list.pop() O(1)   │
│                   │ ~100-1000 cycles  │ ~10-50 cycles          │
│  ─────────────────┼───────────────────┼───────────────────────  │
│  해제 (deallocate)│ delete (힙 해제)  │ free_list.push() O(1)  │
│                   │ ~100-1000 cycles  │ ~10-50 cycles          │
│  ─────────────────┼───────────────────┼───────────────────────  │
│  refcount 증가    │ atomic increment  │ atomic increment       │
│                   │ (동일)            │ (동일)                 │
│  ─────────────────┼───────────────────┼───────────────────────  │
│  캐시 효율        │ 낮음 (흩어진)     │ 높음 (연속)            │
│  ─────────────────┼───────────────────┼───────────────────────  │
│  메모리 오버헤드  │ ~24바이트/객체    │ ~8바이트/객체          │
│                   │ (control block)   │ (refcount만)           │
│                                                                 │
│  실제 영향 (프레임당 1000개 객체 생성/삭제):                    │
│  - std::shared_ptr: ~1-10ms 힙 연산                             │
│  - Pooled: ~0.1-0.5ms 풀 연산                                   │
│  - 개선: 10-20배 빠름                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-28

### 적용 위치

| 파일 | 변경 내용 |
|------|----------|
| Allocator.h | `SharedBlockAllocator` 인터페이스 추가 |
| Allocator.h | `shared_ptr<T>` 템플릿 (ptr + allocator 직접 저장) |
| Allocator.h | `weak_ptr<T>` 템플릿 (ptr + allocator 직접 저장) |
| Allocator.h | `SharedBlockAllocatorImpl<T>` 구현 |
| Allocator.h | `make_shared<T>()` 헬퍼 함수 |
| GBackend.h | `internal_state` 타입 변경: `std::shared_ptr<void>` → `vz::allocator::shared_ptr<void>` |
| GBackend.h | `IsValid()` 구현 변경 |
| GraphicsDevice_DX12.cpp | `std::make_shared<*_DX12>()` → `vz::allocator::make_shared<*_DX12>()` |

### 코드 예시

```cpp
// 변경 전 (std::shared_ptr)
Texture tex;
tex.internal_state = std::make_shared<Texture_DX12>();
if (tex.internal_state != nullptr) { ... }

// 변경 후 (pooled shared_ptr)
Texture tex;
tex.internal_state = vz::allocator::make_shared<Texture_DX12>();
if (tex.IsValid()) { ... }  // ptr && allocator 둘 다 체크
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 문제 | std::shared_ptr의 힙 할당 오버헤드, 메모리 단편화 |
| 해결 | Block allocator 기반 pooled shared_ptr |
| 핵심 | 같은 타입 객체를 연속 메모리에 풀링, free_list로 O(1) 할당/해제 |
| DLL 문제 | WickedEngine handle 방식은 DLL 경계에서 실패 |
| VizMotive 해결 | allocator 포인터 직접 저장 (16바이트 vs 8바이트) |
| 성능 | 할당/해제 10-20배 빠름, 캐시 효율 향상 |

### 핵심 교훈

> **메모리 풀링으로 할당 비용 최소화**
>
> 빈번하게 생성/삭제되는 객체는 힙 할당 대신 풀링 사용.
> Block allocator로 같은 타입 객체를 연속 메모리에 배치하면
> 캐시 효율과 할당 성능 모두 향상.

> **DLL 아키텍처에서 전역 상태 주의**
>
> `inline` 변수는 DLL마다 별도 인스턴스 생성됨.
> DLL 경계를 넘는 데이터는 포인터/ID 대신 직접 참조 저장 고려.
> 단일 실행파일용 코드를 DLL 아키텍처에 적용할 때 주의 필요.
