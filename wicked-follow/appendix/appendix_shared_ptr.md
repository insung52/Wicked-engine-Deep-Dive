# Appendix: shared_ptr / make_shared / weak_ptr

## shared_ptr란

C++에서 객체를 힙에 생성하면 수명을 직접 관리해야 한다.

```cpp
// 일반 포인터 (raw pointer)
Texture_DX12* tex = new Texture_DX12();
// ... 사용 ...
delete tex;  // ← 직접 해제. 까먹으면 메모리 누수, 중복 호출하면 크래시
```

`std::shared_ptr`는 이 수명 관리를 자동화한다.
내부에 **참조 카운트(refcount)**를 보관하고, 마지막 참조가 사라지면 자동으로 `delete` 호출.

```cpp
auto tex = std::make_shared<Texture_DX12>();
// tex를 복사할 때마다 refcount++
// tex가 범위를 벗어나거나 reset() 호출 시 refcount--
// refcount가 0이 되면 자동으로 ~Texture_DX12() + delete
```

---

## refcount 동작 예시

```
make_shared<T>()    → 생성, refcount = 1
shared_ptr<T> b = a → 복사 (공동 소유), refcount = 2
a.reset()           → refcount = 1
b.reset()           → refcount = 0 → ~T() 호출, 메모리 해제
```

그래픽스 엔진에서 유용한 이유:

```cpp
struct Texture
{
    std::shared_ptr<void> internal_state;  // DX12 내부 상태 보유
};

// Texture 객체가 소멸될 때 internal_state도 소멸
// → refcount 0이면 Texture_DX12 자동 해제
// → 엔진이 수명 추적을 직접 하지 않아도 됨
```

---

## make_shared와 control block

```cpp
auto p = std::make_shared<Texture_DX12>();
```

`make_shared`가 힙에 만드는 것:

```
┌──────────────────────────────────────┐  ← 한 번의 힙 할당
│ Control Block                        │
│  atomic<int> strong_count  (4바이트) │  ← refcount
│  atomic<int> weak_count    (4바이트) │  ← weak_ptr 카운트
│  deleter 등                (~8바이트)│
├──────────────────────────────────────┤
│ Texture_DX12 객체           sizeof T │
└──────────────────────────────────────┘

오버헤드: ~24바이트 + 힙 할당 비용
```

---

## 그래픽스 엔진에서의 문제

```
매 프레임 수십~수백 개의 GPU 리소스가 생성/삭제:

CreateTexture() → make_shared<Texture_DX12>() → OS에 메모리 요청 (힙 할당)
CreateBuffer()  → make_shared<Resource_DX12>() → OS에 메모리 요청 (힙 할당)
리소스 해제 시  → refcount 0 → OS에 메모리 반납 (힙 해제)

문제 1: 힙 할당/해제 비용 (~100-1000 cycles/회)
문제 2: 메모리 단편화

  ┌─────────┐  ???  ┌─────────┐  ???  ┌─────────┐
  │Texture A│       │Texture B│       │Texture C│   ← 흩어진 객체들
  └─────────┘       └─────────┘       └─────────┘
      힙 할당 ①         힙 할당 ②         힙 할당 ③
                  캐시 미스 발생!
```

→ 해결: Block Allocator로 같은 타입을 연속 메모리에 풀링
(→ [appendix_block_allocator.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_block_allocator.md))

---

## weak_ptr란

`shared_ptr`는 객체를 **소유**한다 (refcount 증가).
`weak_ptr`는 객체를 **관찰**만 한다 (refcount 증가 없음, 수명에 영향 없음).

```cpp
shared_ptr<T> strong = make_shared<T>();  // refcount = 1
weak_ptr<T>   weak   = strong;            // refcount = 1 그대로 (증가 없음)

strong.reset();  // refcount = 0 → ~T() 호출, 객체 파괴

// 나중에 weak에서 사용 시도:
shared_ptr<T> locked = weak.lock();
if (locked.IsValid())
    // 객체 아직 살아있음
else
    // 객체 이미 파괴됨
```

### lock()의 올바른 구현 — CAS

`lock()`의 핵심: "refcount가 0이 아닐 때만 원자적으로 증가"
단순히 inc → check → dec 순서로 하면 멀티스레드에서 race condition이 생긴다:

```
잘못된 구현 (#9):              올바른 구현 (#19):

inc_refcount()                 try_inc_refcount()  ← CAS
  refcount: 0 → 1               do {
                                   if (refcount == 0) return false;
[다른 스레드: 메모리 재사용]       } while (!CAS(expected, expected+1));
                                 → 체크와 증가를 한 번의 원자 연산으로 처리
dec_refcount()
  refcount: 1 → 0             → 실패 시 아무것도 건드리지 않음
  ↑ 다른 객체의 refcount 오염!
```

