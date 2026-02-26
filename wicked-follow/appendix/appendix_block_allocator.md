# Appendix: Block Allocator / 메모리 풀링

## 힙 할당이란

C++에서 `new` / `malloc`은 OS에 메모리를 요청하는 연산이다.

```cpp
Texture_DX12* p = new Texture_DX12();  // OS에 메모리 요청
// ...
delete p;                               // OS에 메모리 반환
```

### 왜 비싼가

```
new 호출 시:
  1. OS가 사용 가능한 메모리 블록 탐색
  2. 해당 블록을 "사용 중"으로 표시
  3. 포인터 반환

delete 호출 시:
  1. 블록을 "사용 가능"으로 표시
  2. 인접 블록과 병합 (compaction)

비용: ~100-1000 CPU cycles / 회
```

매 프레임 수십~수백 개의 GPU 리소스가 생성/삭제되면, 이 비용이 누적된다.

### 메모리 단편화

```
최초 할당:
  [ A ][ B ][ C ][ D ][ E ][ F ]

B, D, F 해제 후:
  [ A ][   ][ C ][   ][ E ][   ]
        ↑         ↑         ↑
    빈 공간     빈 공간    빈 공간

큰 객체 G 할당 시도:
  → 각 빈 공간이 작아서 G를 못 넣음 (연속 공간 부족)
  → 실패 또는 OS에 더 큰 블록 요청

결과: 메모리는 남아있지만 쓸 수 없는 상태
```

---

## Block Allocator란

같은 타입의 객체를 연속된 메모리 블록에 미리 준비해두는 방식.

```
Block (256개 슬롯):
  ┌────┬────┬────┬────┬────┬────┬────┬────┬─────────────────┐
  │slot│slot│slot│slot│slot│slot│slot│slot│ ... (256개)     │
  │  0 │  1 │  2 │  3 │  4 │  5 │  6 │  7 │                 │
  └────┴────┴────┴────┴────┴────┴────┴────┴─────────────────┘
   ↑
   이 메모리 블록은 한 번의 new RawStruct[256] 으로 통째로 할당
```

객체 할당/해제는 이 블록 안에서만 이루어진다.
OS를 거치지 않으므로 빠르다.

---

## free_list 작동 방식

"현재 비어있는 슬롯 목록"을 스택(stack)으로 관리한다.

```
초기 상태 (256개 모두 비어있음):
  free_list = [slot0, slot1, slot2, ..., slot255]  ← 뒤에서 꺼냄

객체 A 할당:
  slot = free_list.back();  // slot255 꺼냄  O(1)
  free_list.pop_back();
  new (slot) T();            // 생성자 호출 (메모리 할당 없음)
  → free_list = [slot0, ..., slot254]

객체 B 할당:
  slot = free_list.back();  // slot254 꺼냄  O(1)
  free_list.pop_back();
  → free_list = [slot0, ..., slot253]

객체 A 해제:
  ~T();                     // 소멸자 호출 (메모리 반환 없음)
  free_list.push_back(slot) // slot255 다시 넣음  O(1)
  → free_list = [slot0, ..., slot253, slot255]

객체 C 할당:
  slot = free_list.back();  // slot255 재사용  O(1)
  → OS 호출 없음!
```

256개를 모두 소진하면 새 블록을 추가한다 (또 한 번의 `new RawStruct[256]`).

---

## placement new

Block Allocator 안에서 객체를 "생성"할 때는 메모리 할당 없이 생성자만 호출해야 한다.
이것이 `placement new`이다.

```cpp
// 일반 new: 메모리 할당 + 생성자 호출
T* p = new T();

// placement new: 메모리 할당 없음, 생성자만 호출
void* slot = free_list.back();
T* p = new (slot) T();  // ← slot 위치(이미 확보된 메모리)에서 생성자 호출
```

소멸도 마찬가지:
```cpp
// 일반 delete: 소멸자 호출 + 메모리 반환
delete p;

// placement delete: 소멸자만 호출 (메모리 반환 없음)
p->~T();
free_list.push_back(slot);  // 슬롯을 free_list로 직접 반환
```

---

## Block Allocator 구현 예시: RawStruct

### 슬롯 하나의 구조

refcount를 별도 힙에 두지 않고 슬롯 안에 함께 넣으면 control block 할당이 필요 없다.

```cpp
struct alignas(256) RawStruct
{
    uint8_t                data[sizeof(T)];   // T 객체가 들어갈 공간 (placement new 대상)
    std::atomic<uint32_t>  refcount;          // shared_ptr 참조 카운트
    std::atomic<uint32_t>  refcount_weak;     // weak_ptr 참조 카운트
};
```

메모리 레이아웃 (슬롯 2개 연속):

```
주소 0x1000:
┌──────────────────────────────────────────────────────┐
│  data[sizeof(Texture_DX12)]   ← Texture_DX12 객체    │
│  ...                                                  │
├──────────────────────────────────────────────────────┤
│  refcount       (4바이트)                             │
│  refcount_weak  (4바이트)                             │
└──────────────────────────────────────────────────────┘
주소 0x1100 (256바이트 뒤):
┌──────────────────────────────────────────────────────┐
│  data[sizeof(Texture_DX12)]                           │
│  ...                                                  │
├──────────────────────────────────────────────────────┤
│  refcount       (4바이트)                             │
│  refcount_weak  (4바이트)                             │
└──────────────────────────────────────────────────────┘
```

---

### alignas(256)이 하는 일 — 포인터 하위 8비트 트릭

#### 포인터 하위 8비트에 ID를 숨기는 방법

`alignas(256)`은 구조체가 **256의 배수 주소**에만 배치되도록 강제한다.

```
alignas(256) 적용 시:

슬롯 0 주소: 0x1000   =  0001 0000  0000 0000  (이진수)
슬롯 1 주소: 0x1100   =  0001 0001  0000 0000
슬롯 2 주소: 0x1200   =  0001 0010  0000 0000
                                    ─────────
                                    주소값의 하위 8비트 = 항상 0x00
```

왜 하위 8비트가 항상 0인가:

> **주소값이란**: 메모리 주소는 "RAM의 몇 번째 바이트"를 가리키는 **그냥 숫자**다.
> 주소 256은 "256번째 바이트"를 뜻하며, 이진수로 바꾸면 `1_0000_0000` (9비트).

256 = 0x100 = 2^8 이다.
"256의 배수" = "2^8의 배수" = 이진수로 표현했을 때 **하위 8자리가 반드시 0**.

```
256 × 1 = 0x0100  →  0000 0001  [00000000]
256 × 2 = 0x0200  →  0000 0010  [00000000]
256 × 16 = 0x1000 →  0001 0000  [00000000]
256 × 17 = 0x1100 →  0001 0001  [00000000]
                                 ──────────
                                 괄호 안 = 하위 8비트, 항상 0
```

WickedEngine은 이 "항상 0인 8비트"를 버리지 않고 allocator ID 저장에 활용한다:

```cpp
// WickedEngine shared_ptr (8바이트, 단일 uint64_t)
uint64_t handle = (uint64_t(ptr) & ~0xFF) | allocator_id;
//                 └── 상위 56비트: 실제 포인터   └── 하위 8비트: ID

// 복원할 때
T*   ptr          = (T*)(handle & ~0xFF);  // 하위 8비트 0으로 마스킹 → 원래 포인터
int  allocator_id =      handle &  0xFF;  // 하위 8비트만 → ID
```

이렇게 8바이트 하나로 포인터와 ID를 함께 저장할 수 있다.
`alignas(256)`은 **이 트릭의 전제 조건**이다.

#### 대안: allocator 포인터를 별도 멤버로 저장

포인터 트릭을 쓰지 않고, shared_ptr 자체에 allocator 포인터를 저장하는 방법도 있다:

```cpp
// 트릭 방식: 8바이트, 전역 배열 의존
struct shared_ptr { uint64_t handle; };  // 상위 56비트: ptr, 하위 8비트: ID

// 직접 저장 방식: 16바이트, 전역 배열 불필요
struct shared_ptr {
    T*                    ptr;
    SharedBlockAllocator* allocator;
};
```

이 경우 `alignas(256)`은 캐시 라인 정렬 목적으로는 여전히 유효하다.

> CPU는 메모리를 64바이트 단위(캐시 라인)로 읽어온다.
> 256 정렬 시 하나의 슬롯이 여러 캐시 라인에 걸치지 않아 읽기 효율이 높아진다.

> 두 방식 중 어떤 것을 선택할지: [appendix_dll_boundary.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_dll_boundary.md)

---

### refcount를 슬롯 안에 내장하는 이유

`std::shared_ptr`는 refcount를 **control block**이라는 별도 구조에 보관한다.

```
std::make_shared<T>() 내부:

  힙 할당 ①:
  ┌────────────────────────────┐
  │ control block              │  ← refcount, deleter 등
  ├────────────────────────────┤
  │ T 객체                     │  ← 데이터
  └────────────────────────────┘

  shared_ptr 구조:
  ┌──────────────┬──────────────┐
  │    T* ptr    │ ctrl_block*  │  ← 16바이트
  └──────────────┴──────────────┘
```

`make_shared`는 control block + T 객체를 한 번의 힙 할당으로 붙여서 만든다.
그래도 **힙 할당 자체**는 여전히 발생한다.

Block Allocator의 RawStruct는 다르다:

```
make_shared<T>() 내부:

  free_list에서 이미 확보된 슬롯 꺼냄 (힙 할당 없음):
  ┌────────────────────────────┐
  │ data[sizeof(T)]            │  ← 데이터 (placement new로 생성자만 호출)
  ├────────────────────────────┤
  │ refcount                   │  ← control block 역할을 슬롯이 대신
  │ refcount_weak              │
  └────────────────────────────┘

  shared_ptr 구조:
  ┌──────────────┬──────────────┐
  │    T* ptr    │ allocator*   │  ← 16바이트
  └──────────────┴──────────────┘
       ↑ 슬롯의 data 시작 주소     allocator가 refcount 위치를 알고 있음
```

refcount를 접근할 때:
```cpp
// std::shared_ptr: ctrl_block* 역참조
ctrl_block->refcount++;

// Block Allocator: ptr에서 RawStruct로 역산 (같은 슬롯 안에 있으므로)
RawStruct* raw = reinterpret_cast<RawStruct*>(ptr);
raw->refcount++;
```

결론: **슬롯 자체가 control block을 내장**하므로 별도 힙 할당 없음.

---

## 전체 흐름 요약

```
make_shared<T>() 호출:
  1. locker.lock()
  2. free_list.back() → 슬롯 가져오기
  3. free_list.pop_back()
  4. locker.unlock()
  5. new (slot) T()      ← placement new (메모리 할당 없음, 생성자만)
  6. slot.refcount = 1, slot.refcount_weak = 1
  7. shared_ptr { ptr = slot, allocator = this } 반환

shared_ptr 복사:
  allocator->inc_refcount(ptr)  → refcount++  O(1)

shared_ptr 소멸:
  allocator->dec_refcount(ptr)
    refcount-- → 0?
      → ptr->~T()              ← 소멸자 호출 (메모리 반환 없음)
      → dec_refcount_weak(ptr)
          refcount_weak-- → 0?
            → reclaim(ptr)     ← free_list.push_back(slot)
            → 슬롯 재사용 준비 완료
```

---

## Block vs Heap Allocator

상황에 따라 두 가지 방식을 선택한다:

| | Block Allocator (`SharedBlockAllocatorImpl<T, 256>`) | Heap Allocator (`SharedHeapAllocator<T>`) |
|---|---|---|
| 메모리 관리 | 풀링 (256슬롯 블록) | 개별 힙 할당 (`new`/`delete`) |
| 적합한 대상 | 빈번하게 생성/삭제되는 객체 | 프로그램 수명 내내 1개만 존재하는 객체 |
| 생성 함수 예시 | `make_shared<T>()` | `make_shared_single<T>()` |

단일 객체에 블록 풀링을 쓰면 256슬롯 중 1개만 사용하고 나머지가 낭비된다.
→ 이런 싱글턴 객체에는 힙 직접 할당이 오히려 효율적.

> 구체적으로 어떤 타입에 어느 방식을 쓰는지: [apply_allocator_shared_ptr.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/apply_allocator_shared_ptr.md)
