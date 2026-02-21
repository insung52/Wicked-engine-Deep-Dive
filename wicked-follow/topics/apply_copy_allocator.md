# VizMotive 적용: CopyAllocator 개선

## 개요

WickedEngine의 CopyAllocator 개선 과정(topic_copy_allocator.md)을 참고하여,
VizMotive의 CopyAllocator를 **최종 형태**로 직접 수정한 기록.

---

## CopyAllocator란?

CPU 메모리에 있는 데이터(텍스처, 버퍼)를 **GPU 메모리(VRAM)로 업로드**하는 전용 시스템.

```
[CPU 메모리]                         [GPU 메모리 (VRAM)]
     │                                    ▲
     ▼                                    │
[Upload Buffer]  ──CopyAllocator 큐──▶  [Resource]
 (UPLOAD heap)      (COPY 타입)        (DEFAULT heap)
```

사용 시점: `CreateBuffer()`, `CreateTexture()` 등 **리소스 생성 시 초기 데이터 업로드**.

---

## 배경 지식: 큐/커맨드 관련 함수 정리

### ExecuteCommandLists vs SubmitCommandLists vs queue.submit

| 함수 | 레벨 | 역할 |
|------|------|------|
| `ID3D12CommandQueue::ExecuteCommandLists()` | DX12 API | 커맨드 리스트를 GPU 큐에 **직접 제출** |
| `CommandQueue::submit()` | 엔진 내부 | `submit_cmds`에 모인 커맨드 리스트를 `ExecuteCommandLists()`로 **일괄 제출** |
| `GraphicsDevice_DX12::SubmitCommandLists()` | 엔진 최상위 | 프레임 전체의 모든 큐 submit + fence signal + present + CPU wait |

관계:

```
SubmitCommandLists()                    ← 프레임 끝에 1번 호출
  └─ queue.submit()                     ← 각 큐별로 호출
       └─ queue->ExecuteCommandLists()  ← DX12 API
```

CopyAllocator는 프레임 루프와 무관하게 **`ExecuteCommandLists()`를 직접 호출**함.
"지금 복사해 → 기다려 → 끝" 흐름을 단독으로 처리하기 때문.

---

## 배경 지식: CopyAllocator 큐 vs 메인 QUEUE_COPY

### 두 개의 Copy 타입 큐가 존재하는 이유

CopyAllocator는 메인 `QUEUE_COPY`를 쓰지 않고 **자기만의 Copy Queue를 별도로 생성**함.
타입은 `D3D12_COMMAND_LIST_TYPE_COPY`로 같지만 **별개의 큐 인스턴스**임.

```cpp
// CopyAllocator::init()
#ifdef PLATFORM_XBOX
    queue = device->queues[QUEUE_COPY].queue;  // XBOX: 큐 생성 제한 → 메인 큐 공유
#else
    // PC: 별도 Copy Queue 생성
    D3D12_COMMAND_QUEUE_DESC desc = {};
    desc.Type = D3D12_COMMAND_LIST_TYPE_COPY;
    device->device->CreateCommandQueue(&desc, PPV_ARGS(queue));
#endif
```

### 왜 따로 만들었나

핵심: **CopyAllocator는 프레임 루프 바깥에서 호출되기 때문.**

| | 메인 QUEUE_COPY | CopyAllocator 큐 |
|---|---|---|
| 관리 주체 | `SubmitCommandLists()` | CopyAllocator 자체 |
| 제출 시점 | 프레임 끝에 일괄 제출 | 아무 타이밍에 즉시 제출 |
| 동기화 | frame_fence | 자체 fence + CPU wait |
| 용도 | 프레임 내 복사 작업 | 리소스 생성 시 초기 데이터 업로드 |

메인 큐를 공유하면 충돌 발생:

```
스레드 A (렌더링):  QUEUE_COPY에 프레임 커맨드 쌓는 중...
스레드 B (로딩):    CopyAllocator가 같은 QUEUE_COPY에 ExecuteCommandLists() 호출!
                    → 큐 하나에 두 곳에서 동시에 제출 → 충돌/간섭
```

별도 큐면 GPU가 병렬 처리 가능:

```
메인 QUEUE_COPY:     [프레임 내 복사]     ← SubmitCommandLists()가 관리
CopyAllocator 큐:    [텍스처 로딩]        ← 독립적으로 즉시 실행 + CPU 대기
                GPU가 두 큐를 병렬 처리
```

### VizMotive에서의 실제 사용 코드

**CopyAllocator 큐** — 리소스 생성 시 (GraphicsDevice_DX12.cpp):

```cpp
// CreateBuffer() 내부 (3487줄) — 버퍼에 초기 데이터 업로드
cmd = copyAllocator.allocate(desc->size);
cmd.commandList->CopyBufferRegion(dest, 0, src, 0, size);
copyAllocator.submit(cmd);  // 별도 큐에서 즉시 실행 + CPU 대기

// CreateTexture() 내부 (3880줄) — 텍스처에 초기 데이터 업로드
cmd = copyAllocator.allocate(internal_state->total_size);
cmd.commandList->CopyTextureRegion(&Dst, 0, 0, 0, &Src, nullptr);
copyAllocator.submit(cmd);  // 별도 큐에서 즉시 실행 + CPU 대기
```

**메인 QUEUE_COPY** — 프레임 렌더링 시:

```cpp
// Renderer.cpp:385 — 현재 주석 처리됨 (사용 안 함)
//device->WaitQueue(cmd, QUEUE_TYPE::QUEUE_COPY);
```

현재 VizMotive에서는 메인 QUEUE_COPY를 렌더링 중에 직접 사용하는 코드가 없음.
GRAPHICS 큐가 복사 명령도 지원하므로, 프레임 내 복사는 GRAPHICS 큐에서 처리.

---

## freelist란?

`CopyCMD` 객체(커맨드리스트 + 업로드 버퍼 + fence)를 **재사용하기 위한 풀**.

```
최초 요청:   allocate() → freelist 비어있음 → 새로 생성
             submit()  → 복사 완료 후 freelist에 반환

두번째 요청: allocate() → freelist에서 크기 맞는 거 꺼냄 → 재사용!
             submit()  → 복사 완료 후 freelist에 반환
```

GPU 리소스(커맨드리스트, fence, 업로드 버퍼) 생성이 비싸므로 한 번 만든 것을 계속 돌려씀.

---

## 변경 전 상태 (VizMotive 원본)

### CopyCMD 구조체 (GraphicsDevice_DX12.h)

```cpp
struct CopyCMD
{
    ComPtr<ID3D12CommandAllocator> commandAllocator;
    ComPtr<ID3D12GraphicsCommandList> commandList;
    ComPtr<ID3D12Fence> fence;
    uint64_t fenceValueSignaled = 0;  // 매번 증가
    GPUBuffer uploadbuffer;
    inline bool IsValid() const { return commandList != nullptr; }
    inline bool IsCompleted() const { return fence->GetCompletedValue() >= fenceValueSignaled; }
};
```

### submit 함수

```cpp
void CopyAllocator::submit(CopyCMD cmd)
{
    locker.lock();
    cmd.fenceValueSignaled++;           // 값 증가
    freelist.push_back(cmd);            // 실행 전에 freelist에 추가 (미완료 상태!)
    locker.unlock();

    cmd.commandList->Close();
    queue->ExecuteCommandLists(1, commandlists);
    queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled);

    // 모든 GPU 큐가 복사 완료를 대기 (GPU Wait 방식)
    device->queues[QUEUE_GRAPHICS].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
    device->queues[QUEUE_COMPUTE].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
    device->queues[QUEUE_COPY].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
    device->queues[QUEUE_VIDEO_DECODE].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
}
```

### allocate 함수

```cpp
for (size_t i = 0; i < freelist.size(); ++i)
{
    if (freelist[i].uploadbuffer.desc.size >= staging_size)
    {
        if (freelist[i].IsCompleted())  // 매번 완료 체크 필요
        {
            cmd = std::move(freelist[i]);
            // ...
```

### 문제점

1. `fenceValueSignaled`가 계속 증가 → 복잡한 상태 관리
2. freelist에 **미완료** cmd 존재 → `IsCompleted()` 매번 호출 필요
3. **GPU Wait 방식** → 모든 메인 큐가 allocator copy 큐에 의존, 데드락 위험

---

## 변경 후 상태

### CopyCMD 구조체 변경 (GraphicsDevice_DX12.h)

```cpp
struct CopyCMD
{
    ComPtr<ID3D12CommandAllocator> commandAllocator;
    ComPtr<ID3D12GraphicsCommandList> commandList;
    ComPtr<ID3D12Fence> fence;
    // fenceValueSignaled 제거 — 고정 0/1 패턴 사용
    GPUBuffer uploadbuffer;
    inline bool IsValid() const { return commandList != nullptr; }
    // IsCompleted() 제거 — freelist에 있으면 항상 완료 상태
};
```

### submit 함수 변경

```cpp
void CopyAllocator::submit(CopyCMD cmd)
{
    dx12_check(cmd.commandList->Close());
    ID3D12CommandList* commandlists[] = { cmd.commandList.Get() };

    dx12_check(cmd.fence->Signal(0));               // CPU에서 fence를 0으로 리셋
    queue->ExecuteCommandLists(1, commandlists);     // GPU 복사 실행
    dx12_check(queue->Signal(cmd.fence.Get(), 1));   // GPU 완료 시 fence를 1로 signal

    // GPU Wait 제거 → CPU Wait로 변경
    dx12_check(cmd.fence->SetEventOnCompletion(1, nullptr));  // CPU가 완료까지 대기

    // 완료 후에만 freelist에 추가 (완료 보장)
    std::scoped_lock lock(locker);
    freelist.push_back(cmd);
}
```

### allocate 함수 변경

```cpp
for (size_t i = 0; i < freelist.size(); ++i)
{
    if (freelist[i].uploadbuffer.desc.size >= staging_size)
    {
        // IsCompleted() 체크 제거 — freelist에 있으면 항상 완료 상태
        cmd = std::move(freelist[i]);
        std::swap(freelist[i], freelist.back());
        freelist.pop_back();
        break;
    }
}
```

---

## 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| fence 값 | `fenceValueSignaled++` (무한 증가) | 고정 0 → 1 |
| 동기화 방식 | GPU Wait (모든 메인 큐가 대기) | CPU Wait (CPU만 대기) |
| 완료 체크 | `IsCompleted()` 매번 호출 | 불필요 (freelist = 완료됨) |
| freelist 추가 시점 | 실행 전 (미완료 상태) | 실행 완료 후 |
| 에러 체크 | assert만 | dx12_check 추가 |

---

## 주의: fence->Signal vs queue->Signal

같은 이름이지만 **호출 대상에 따라 완전히 다른 동작**:

| 호출 | 실행 시점 | 용도 |
|------|-----------|------|
| `cmd.fence->Signal(0)` | **CPU에서 즉시** 값 변경 | fence 리셋 |
| `queue->Signal(cmd.fence.Get(), 1)` | **GPU가 나중에** 큐의 이전 작업 완료 후 실행 | 복사 완료 signal |

```cpp
dx12_check(cmd.fence->Signal(0));              // ① CPU: 즉시 fence = 0 (리셋)
queue->ExecuteCommandLists(1, commandlists);   // ② GPU: 복사 작업 시작
dx12_check(queue->Signal(cmd.fence.Get(), 1)); // ③ GPU: 복사 끝나면 fence = 1
dx12_check(cmd.fence->SetEventOnCompletion(1, nullptr)); // ④ CPU: fence가 1이 될 때까지 대기
```

---

## 주의: std::scoped_lock

`std::scoped_lock`은 스코프(`{}`) 벗어나면 **자동으로 unlock**.
수동 unlock 불필요, 예외 발생 시에도 안전.

```cpp
{
    std::scoped_lock lock(locker);  // lock
    freelist.push_back(cmd);
}  // 자동 unlock (소멸자)
```

---

## GPU Wait vs CPU Wait 비교

```
[GPU Wait 방식 — 변경 전]

CopyAllocator 큐:  [████복사████] ──Signal──▶
                                      │
GRAPHICS 큐:       ◀──Wait────────────┘ [렌더링]  (의존성!)
COMPUTE 큐:        ◀──Wait────────────┘ [계산]    (의존성!)

문제: 모든 메인 GPU 큐가 복사에 의존 → 데드락 위험, 복잡한 큐 간 의존성

[CPU Wait 방식 — 변경 후]

CopyAllocator 큐:  [████복사████] ──Signal──▶
                                      │
CPU:               [대기..........] ◀─┘ [다음 작업]

GRAPHICS 큐:       [렌더링]  (독립적)
COMPUTE 큐:        [계산]    (독립적)

장점: GPU 큐 간 의존성 없음, 단순함
단점: CPU 블로킹 (리소스 로딩 시점이라 체감 없음)
```
