# 커밋 #22: 43b93085 - dx12, vulkan: using block allocator for command lists

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `43b93085` |
| 날짜 | 2026-01-15 |
| 작성자 | Turánszki János |
| 카테고리 | 최적화 |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.h | `cmd_allocator` + raw pointer 벡터로 변경 |
| wiGraphicsDevice_DX12.cpp | BlockAllocator로 CommandList 풀링 |

---

## 배경 지식: CommandList와 메모리 관리

### CommandList란?

```
┌─────────────────────────────────────────────────────────────────┐
│              DX12 CommandList 개념                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CommandList = GPU에 보낼 명령들의 리스트                       │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  CommandList 내용 예시:                                   │  │
│  │                                                           │  │
│  │  1. SetPipelineState(pso)         ← 파이프라인 설정       │  │
│  │  2. SetRenderTargets(rtv)         ← 렌더타겟 설정         │  │
│  │  3. SetVertexBuffers(vb)          ← 버텍스 버퍼 바인딩    │  │
│  │  4. DrawIndexed(36, 0, 0)         ← 드로우 호출           │  │
│  │  5. ResourceBarrier(...)          ← 리소스 상태 전환      │  │
│  │  6. Close()                       ← 녹화 완료             │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  CPU에서 명령을 "녹화"하고, GPU로 "제출(Execute)"               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### CommandList 생성/파괴 빈도

```
┌─────────────────────────────────────────────────────────────────┐
│              CommandList 생성 패턴                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  매 프레임마다 여러 CommandList 사용:                           │
│                                                                 │
│  Frame N:                                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ CmdList[0]: Shadow Pass                                 │    │
│  │ CmdList[1]: G-Buffer Pass                               │    │
│  │ CmdList[2]: Lighting Pass                               │    │
│  │ CmdList[3]: Post-Processing                             │    │
│  │ CmdList[4]: UI Rendering                                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  필요에 따라 더 많은 CommandList 생성:                          │
│  - 멀티스레드 렌더링 → 스레드당 CommandList                     │
│  - 동적 작업량 → 필요시 추가 할당                               │
│                                                                 │
│  문제: std::unique_ptr로 개별 할당 시                           │
│  - 매번 힙 할당/해제 오버헤드                                   │
│  - 메모리 단편화                                                │
│  - 캐시 미스 증가                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 문제: unique_ptr 개별 할당의 비효율

### 기존 코드 (변경 전)

```cpp
// 변경 전 - 개별 힙 할당
struct GraphicsDevice_DX12
{
    std::vector<std::unique_ptr<CommandList_DX12>> commandlists;
    uint32_t cmd_count = 0;
};

// CommandList 할당
CommandList GraphicsDevice_DX12::BeginCommandList(QUEUE_TYPE queue)
{
    cmd_locker.lock();
    uint32_t cmd_current = cmd_count++;
    if (cmd_current >= commandlists.size())
    {
        // 💥 매번 새로운 힙 할당!
        commandlists.push_back(std::make_unique<CommandList_DX12>());
    }
    CommandList cmd;
    cmd.internal_state = commandlists[cmd_current].get();
    // ...
}
```

### 문제점

```
┌─────────────────────────────────────────────────────────────────┐
│              std::unique_ptr 개별 할당 문제                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  메모리 레이아웃 (단편화):                                      │
│                                                                 │
│  Heap:                                                          │
│  ┌────┬────┬─────────┬────┬────────┬────┬──────────────────┐    │
│  │Cmd0│????│  Cmd1   │????│  Cmd2  │????│      Cmd3        │    │
│  └────┴────┴─────────┴────┴────────┴────┴──────────────────┘    │
│    ↑    ↑      ↑       ↑      ↑      ↑         ↑                │
│    │    │      │       │      │      │         │                │
│    │  다른 할당들이    │     다른 할당들        │                │
│    │  중간에 삽입됨    │     중간에 삽입됨      │                │
│                                                                 │
│  문제:                                                          │
│  1. 캐시 미스: CommandList들이 메모리에 흩어져 있음             │
│  2. 할당 오버헤드: malloc/free 호출 비용                        │
│  3. 메모리 단편화: 작은 할당들이 여기저기 분산                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 해결: BlockAllocator 풀링

### BlockAllocator 동작 원리

```
┌─────────────────────────────────────────────────────────────────┐
│              BlockAllocator<CommandList_DX12, 64>                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Block 0 (64개 연속 할당):                                      │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬─────┬────┐           │
│  │Cmd │Cmd │Cmd │Cmd │Cmd │Cmd │Cmd │Cmd │ ... │Cmd │           │
│  │ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │     │ 63 │           │
│  └────┴────┴────┴────┴────┴────┴────┴────┴─────┴────┘           │
│  ↑                                                              │
│  연속된 메모리 블록 → 캐시 효율!                                │
│                                                                 │
│  Block 1 (추가 64개 필요 시):                                   │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬─────┬────┐           │
│  │Cmd │Cmd │Cmd │Cmd │Cmd │Cmd │Cmd │Cmd │ ... │Cmd │           │
│  │ 64 │ 65 │ 66 │ 67 │ 68 │ 69 │ 70 │ 71 │     │127 │           │
│  └────┴────┴────┴────┴────┴────┴────┴────┴─────┴────┘           │
│                                                                 │
│  Free List (재사용 목록):                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ [Cmd7] [Cmd6] [Cmd5] [Cmd4] [Cmd3] [Cmd2] [Cmd1] [Cmd0] │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                       ↑         │
│                              allocate() → 여기서 꺼냄           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 변경된 코드 (변경 후)

```cpp
// 변경 후 - BlockAllocator 풀링
struct GraphicsDevice_DX12
{
    BlockAllocator<CommandList_DX12, 64> cmd_allocator;  // ✅ 풀링!
    std::vector<CommandList_DX12*> commandlists;         // ✅ raw pointer
    uint32_t cmd_count = 0;
};

// CommandList 할당
CommandList GraphicsDevice_DX12::BeginCommandList(QUEUE_TYPE queue)
{
    cmd_locker.lock();
    uint32_t cmd_current = cmd_count++;
    if (cmd_current >= commandlists.size())
    {
        // ✅ 풀에서 빠르게 할당 (O(1))
        commandlists.push_back(cmd_allocator.allocate());
    }
    CommandList cmd;
    cmd.internal_state = commandlists[cmd_current];  // ✅ .get() 불필요
    // ...
}
```

---

## BlockAllocator 구현 분석

### allocate() 함수

```cpp
template<typename T, size_t block_size = 256>
struct BlockAllocator
{
    struct Block
    {
        // ✅ T 타입의 alignment 보장
        struct alignas(alignof(T)) RawStruct
        {
            uint8_t data[sizeof(T)];
        };
        std::vector<RawStruct> mem;
    };
    std::vector<Block> blocks;
    std::vector<T*> free_list;

    template<typename... ARG>
    inline T* allocate(ARG&&... args)
    {
        if (free_list.empty())
        {
            // 새 블록 할당 (64개 단위)
            free_list.reserve(block_size);
            Block& block = blocks.emplace_back();
            block.mem.resize(block_size);

            // 모든 슬롯을 free_list에 추가
            T* ptr = (T*)block.mem.data();
            for (size_t i = 0; i < block_size; ++i)
            {
                free_list.push_back(ptr + i);
            }
        }

        // free_list에서 꺼내서 placement new
        T* ptr = free_list.back();
        free_list.pop_back();
        return new (ptr) T(std::forward<ARG>(args)...);  // ✅ placement new
    }
};
```

### free() 함수

```cpp
inline void free(T* ptr)
{
    ptr->~T();                    // 소멸자 호출
    free_list.push_back(ptr);     // 슬롯 재사용 가능하게
}
```

---

## 성능 비교

```
┌─────────────────────────────────────────────────────────────────┐
│              성능 비교: unique_ptr vs BlockAllocator             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  할당 성능:                                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ std::make_unique<T>()                                     │  │
│  │   1. malloc() 호출 (시스템 호출 가능)                     │  │
│  │   2. 메모리 초기화                                        │  │
│  │   3. 생성자 호출                                          │  │
│  │   → 수백~수천 cycles                                      │  │
│  │                                                           │  │
│  │ BlockAllocator::allocate()                                │  │
│  │   1. free_list.pop_back() (O(1))                          │  │
│  │   2. placement new (생성자만 호출)                        │  │
│  │   → 수십 cycles                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  메모리 레이아웃:                                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ unique_ptr: 흩어진 할당 → L1/L2 캐시 미스 다수            │  │
│  │ BlockAllocator: 연속 메모리 → 캐시 히트율 높음            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  예상 성능 향상:                                                │
│  - 할당/해제: 10~20배 빠름                                      │
│  - 순회 시 캐시 효율: 2~5배 향상                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 소멸자 정리 코드 (VizMotive 추가)

### WickedEngine에 없는 정리 코드

WickedEngine 원본 코드에는 소멸자에서 commandlists 정리 코드가 없음.
VizMotive에서는 명시적으로 정리 필요:

```cpp
GraphicsDevice_DX12::~GraphicsDevice_DX12()
{
    WaitForGPU();  // GPU 작업 완료 대기

    // ✅ BlockAllocator로 할당한 commandlist 정리
    for (auto* cmdlist : commandlists)
    {
        cmd_allocator.free(cmdlist);  // 소멸자 호출 + 슬롯 반환
    }
    commandlists.clear();

    // ... 나머지 정리 코드 ...
}
```

### 왜 명시적 정리가 필요한가?

```
┌─────────────────────────────────────────────────────────────────┐
│              BlockAllocator 정리 필요성                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  unique_ptr 사용 시:                                            │
│  - 벡터 소멸 → unique_ptr 소멸 → 자동으로 delete 호출          │
│  - 명시적 정리 불필요                                           │
│                                                                 │
│  raw pointer + BlockAllocator 사용 시:                          │
│  - 벡터 소멸 → 포인터만 사라짐, 객체는 그대로                   │
│  - 소멸자 호출 안 됨 → 리소스 누수!                             │
│  - 명시적으로 cmd_allocator.free() 호출 필요                    │
│                                                                 │
│  ⚠️ BlockAllocator 자체가 소멸될 때:                            │
│  - blocks 벡터가 메모리 해제                                    │
│  - 하지만 T::~T() 소멸자는 호출 안 됨!                          │
│  - CommandList_DX12는 COM 객체 등 보유 → 반드시 소멸자 필요     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 다이어그램: 전체 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│              CommandList 생명주기                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. BeginCommandList() 호출                                     │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ cmd_allocator.allocate()                            │     │
│     │   ↓                                                 │     │
│     │ free_list에서 슬롯 할당 (또는 새 블록 생성)         │     │
│     │   ↓                                                 │     │
│     │ placement new로 CommandList_DX12 생성               │     │
│     │   ↓                                                 │     │
│     │ commandlists 벡터에 포인터 저장                     │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
│  2. 렌더링 중 (CommandList 사용)                                │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ 명령 녹화, 제출, GPU 실행                           │     │
│     │ cmd_count 리셋하여 다음 프레임에 재사용             │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
│  3. 디바이스 종료 시                                            │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ WaitForGPU()                                        │     │
│     │   ↓                                                 │     │
│     │ for (cmdlist : commandlists)                        │     │
│     │     cmd_allocator.free(cmdlist)  ← 소멸자 + 반환    │     │
│     │   ↓                                                 │     │
│     │ commandlists.clear()                                │     │
│     │   ↓                                                 │     │
│     │ cmd_allocator 소멸 (블록 메모리 해제)               │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## VizMotive 적용 현황

### 적용 완료 ✅

**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.h | `BlockAllocator<CommandList_DX12, 64> cmd_allocator` 추가 |
| GraphicsDevice_DX12.h | `std::vector<CommandList_DX12*> commandlists` (raw pointer) |
| GraphicsDevice_DX12.cpp | `cmd_allocator.allocate()` 사용 |
| GraphicsDevice_DX12.cpp | `.get()` 제거 (raw pointer 직접 사용) |
| GraphicsDevice_DX12.cpp | 소멸자에 `cmd_allocator.free()` 루프 추가 |

### 코드 위치

```cpp
// GraphicsDevice_DX12.h (라인 254-257)
vz::allocator::BlockAllocator<CommandList_DX12, 64> cmd_allocator;
std::vector<CommandList_DX12*> commandlists;
uint32_t cmd_count = 0;
vz::SpinLock cmd_locker;

// GraphicsDevice_DX12.cpp (라인 3072-3077) - 소멸자
for (auto* cmdlist : commandlists)
{
    cmd_allocator.free(cmdlist);
}
commandlists.clear();

// GraphicsDevice_DX12.cpp (라인 5390-5396) - 할당
if (cmd_current >= commandlists.size())
{
    commandlists.push_back(cmd_allocator.allocate());
}
CommandList cmd;
cmd.internal_state = commandlists[cmd_current];
```

---

## 요약

| 항목 | 내용 |
|------|------|
| 목적 | CommandList 메모리 풀링으로 할당 오버헤드 감소 |
| 변경 전 | `std::unique_ptr<CommandList_DX12>` 개별 힙 할당 |
| 변경 후 | `BlockAllocator<CommandList_DX12, 64>` 블록 풀링 |
| 효과 | 할당 10-20배 빠름, 캐시 효율 향상, 단편화 방지 |
| 주의 | 소멸자에서 명시적 정리 필요 (VizMotive 추가) |
| VizMotive | ✅ 적용 완료 |

### 핵심 교훈

> **같은 타입 객체의 빈번한 할당 → BlockAllocator**
>
> CommandList처럼 같은 타입 객체가 자주 생성/삭제되는 경우
> BlockAllocator 패턴이 효과적:
> - 64개 단위로 연속 메모리 할당
> - free_list로 O(1) 할당/해제
> - 캐시 지역성 향상
>
> 단, raw pointer 사용 시 명시적 소멸자 호출 필수!

> **unique_ptr → raw pointer + 풀링 시 주의사항**
>
> 1. `get()` 호출 제거
> 2. 소멸자에서 명시적 free() 호출
> 3. 블록 크기는 일반적 사용량에 맞게 설정 (64~256)
> 4. 스레드 안전이 필요하면 SpinLock 사용

