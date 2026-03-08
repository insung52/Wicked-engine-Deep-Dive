# VizMotive 적용: CopyAllocator 개선

**[Notebooklm pdf 문서 : copy_allocator](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/pdf/copy_allocator.pdf)**

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

### 레이어 개요

이 엔진에는 "커맨드 리스트"라는 이름이 두 군데 나온다. 레이어를 먼저 구분한다.

```
[최상위 — 셰이더 엔진 레벨]
  CommandList (= 정수 ID)            ← BeginCommandList()로 받는 핸들
  │  내부적으로 CommandList_DX12 = { allocator[BUFFERCOUNT][QUEUE_COUNT]
  │                                  commandList[QUEUE_COUNT]
  │                                  DescriptorBinder, ... }
  │
  ├─ WaitCommandList(cmd_A, cmd_B)   ← GPU 실행 순서 보장 (GPU-side)
  └─ SubmitCommandLists()            ← 프레임 끝에 모든 큐를 GPU에 제출

[중간 — 엔진 DX12 내부]
  CommandQueue::submit()             ← submit_cmds 모아서 ExecuteCommandLists() 호출

[하위 — DX12 API]
  ID3D12CommandQueue (GPU Queue)     ← 실제 GPU 실행 파이프라인
  ID3D12GraphicsCommandList          ← GPU에 전달되는 명령 버퍼
  ID3D12CommandQueue::ExecuteCommandLists()  ← GPU에 직접 제출
```

### ExecuteCommandLists vs SubmitCommandLists vs queue.submit

| 함수 | 레벨 | 역할 |
|------|------|------|
| `device->BeginCommandList()` | 셰이더엔진 | CPU 스레드에서 커맨드 기록 시작, `CommandList` 핸들 반환 |
| `device->WaitCommandList(A, B)` | 셰이더엔진 | **GPU 실행 순서 보장**: A가 실행되기 전 B의 완료를 GPU가 대기 |
| `ID3D12CommandQueue::ExecuteCommandLists()` | DX12 API | 커맨드 리스트를 GPU 큐에 **직접 제출** |
| `CommandQueue::submit()` | 엔진 내부 | `submit_cmds`에 모인 커맨드 리스트를 `ExecuteCommandLists()`로 **일괄 제출** |
| `GraphicsDevice_DX12::SubmitCommandLists()` | 엔진 최상위 | 프레임 전체의 모든 큐 submit + fence signal + present + **CPU wait** |

관계:

```
SubmitCommandLists()                    ← 프레임 끝에 1번 호출 (CPU 레벨에서 대기 포함)
  └─ queue.submit()                     ← 각 큐별로 호출
       └─ queue->ExecuteCommandLists()  ← DX12 API
```

CopyAllocator는 프레임 루프와 무관하게 **`ExecuteCommandLists()`를 직접 호출**함.
"지금 복사해 → 기다려 → 끝" 흐름을 단독으로 처리하기 때문.

### WaitCommandList 메커니즘 — GPU 내부 순서 보장 (GraphicsDevice_DX12.cpp:6091)

```cpp
void GraphicsDevice_DX12::WaitCommandList(CommandList cmd, CommandList wait_for)
{
    Semaphore semaphore = new_semaphore();       // fence + fenceValue 쌍
    commandlist.waits.push_back(semaphore);      // cmd는 이 semaphore를 기다림
    commandlist_wait_for.signals.push_back(semaphore); // wait_for는 완료 시 이 semaphore를 signal
}
```

- `Semaphore = {ID3D12Fence, fenceValue}` 쌍
- **CPU가 기다리는 것이 아님** — GPU 큐 내부에서 cmd가 실행되기 전, wait_for의 signal을 GPU가 체킹
- `SubmitCommandLists()` 내부에서 `queue->Wait(fence, value)` 형태로 GPU 큐에 등록됨

### jobsystem과 WaitCommandList의 관계 (Renderer.cpp)

```cpp
jobsystem::context ctx;
// 여러 커맨드를 CPU 스레드들이 병렬로 기록:
jobsystem::Execute(ctx, [this, cmd](jobsystem::JobArgs args) {
    device->WaitCommandList(cmd, cmd_prepareframe); // GPU 실행 순서 지정
    DrawScene(...);
});
jobsystem::Execute(ctx, [this, cmd](jobsystem::JobArgs args) { ... });
jobsystem::Wait(ctx);  // CPU에서: 모든 기록 스레드 완료 대기
// → SubmitCommandLists() 호출
```

| | jobsystem::Execute/Wait | WaitCommandList |
|---|---|---|
| **작동 레이어** | CPU (스레드 병렬화) | GPU (큐 내 실행 순서) |
| **목적** | 커맨드 기록을 여러 스레드에서 동시에 수행 | GPU가 실행할 때 A가 B보다 나중에 실행되도록 보장 |
| **대기 주체** | CPU 스레드 | GPU 큐 |
| **Fence 사용** | N/A | Semaphore (fence + value) |

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

### WickedEngine에서 메인 QUEUE_COPY를 쓰는 시나리오

GPU 하드웨어에는 독립적으로 병렬 동작하는 전용 엔진들이 있다:

```
GPU 내부:
  3D Engine     ← QUEUE_GRAPHICS 처리
  Compute Engine ← QUEUE_COMPUTE 처리
  DMA Engine    ← QUEUE_COPY 처리  ← 전용 복사 하드웨어
```

QUEUE_COPY는 DMA(Direct Memory Access) 엔진을 사용하므로 GRAPHICS/COMPUTE와 겹쳐서 동시 실행 가능.
> **DMA 엔진이란?** CPU나 GPU의 메인 연산 코어(3D/Compute)와 독립적으로 **메모리 간 데이터 전송(복사)만 전담하는 하드웨어 유닛**이다. 연산 코어의 리소스를 소모하거나 작업 흐름을 방해하지 않고 백그라운드에서 메모리를 복사할 수 있어 병렬 처리에 매우 효과적이다.
WickedEngine은 이 특성을 프레임 내 복사 작업에 활용한다.

**시나리오 1: 텍스처 스트리밍 mip 업로드**
```
QUEUE_GRAPHICS: [저해상도 mip으로 렌더링 중...]
QUEUE_COPY:     [고해상도 mip을 VRAM에 업로드 중...]  ← 동시 진행

다음 프레임:
QUEUE_GRAPHICS: WaitCommandList → 고해상도 mip으로 전환
```

**시나리오 2: 스킨드 메시 버퍼 갱신**
```
QUEUE_COMPUTE:  [본 행렬 계산 → skinned vertex 기록] ─Signal─▶
QUEUE_COPY:     ◀─Wait─ [skinned buffer → render vertex buffer 복사] ─Signal─▶
QUEUE_GRAPHICS: ◀─Wait─ [최종 vertex buffer로 렌더링]
```

**시나리오 3: 파티클 시스템 상태 복사**

매 프레임 GPU 파티클 시뮬레이션 결과(COMPUTE)를 렌더용 버퍼에 복사(COPY).
GRAPHICS는 `WaitCommandList`로 COPY 완료 후 사용.

> VizMotive는 현재 이런 시스템(스트리밍/스킨드 메시/파티클)이 없어서 프레임 내 전용 복사 작업 자체가 없다.
> GRAPHICS 큐가 복사 명령도 지원하므로 별도로 쓸 필요가 없는 상태.
> 큐는 준비되어 있고, 해당 시스템 구현 시 활성화된다.

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
단점: CPU 블로킹
  - 초기 로딩 시: 이미 로딩 화면 등이 보이고 있으므로 체감 없음
  - 런타임 중 동적 로딩 시: 렌더 루프가 CPU wait에서 멈추므로
    유저가 순간적인 프레임 드롭/버벅임을 경험할 수 있음
    → 런타임 로딩은 별도 스레드에서 처리하고, 완료 후 안전한 시점에
      Scene에 반영하는 메커니즘이 필요 (아래 섹션 참고)
```

---

## 런타임 로딩과 비동기 처리 (렌더링 엔진 레벨 feature)

현재 VizMotive의 `CopyAllocator::submit()`은 GPU 복사 완료까지 **CPU를 블로킹**한다.

이 구조는 초기 로딩 시점에는 문제없지만, **앱 실행 중 동적으로 리소스 로딩**이
일어나면 프레임 루프가 멈춰 유저가 버벅임을 경험한다.

### WickedEngine의 해결 방법 1: LoadingScreen (wiLoadingScreen.cpp)

큰 규모 로딩(다음 씬 전환 등)을 위한 패턴:

```cpp
// LoadingScreen::Start() — wiLoadingScreen.cpp:55

// 1. 로딩 작업들을 jobsystem 백그라운드 스레드로 제출:
for (auto& x : tasks)
    wi::jobsystem::Execute(ctx, x);  // CPU 스레드 풀에서 병렬 실행

// 2. 별도 std::thread를 떼어내어 완료를 비동기로 기다림:
std::thread([this]() {
    wi::jobsystem::Wait(ctx);  // 로딩 완료까지 이 스레드만 블로킹

    // 3. 완료되면 메인 스레드가 안전한 시점(프레임 경계)에 콜백 실행:
    wi::eventhandler::Subscribe_Once(wi::eventhandler::EVENT_THREAD_SAFE_POINT,
        [this](uint64_t) {
            finish();  // 로딩 완료 후 씬 전환 등
        });
}).detach();  // 메인 스레드(렌더 루프)에는 영향 없음
```

**`jobsystem::Execute`가 실제로 하는 것**:
각 task(`x`)를 **메인 스레드가 아닌 워커 스레드**에 배분한다.

task 내부에서는 `CreateTexture()` → `copyAllocator.submit()` → `SetEventOnCompletion` (CPU blocking) 이 일어나지만,
그 blocking이 **워커 스레드에서** 일어난다는 것이 핵심이다.

```
메인 스레드 (렌더 루프):   [렌더링...로딩 화면...렌더링...]  ← 전혀 멈추지 않음

워커 스레드 A:  [CreateTexture()] → copyAllocator.submit() → GPU copy → [CPU blocking] → 완료
워커 스레드 B:  [CreateTexture()] → copyAllocator.submit() → GPU copy → [CPU blocking] → 완료
```

**copy 메커니즘 자체(CopyAllocator의 CPU blocking)는 똑같다.**
차이는 그 blocking이 어느 스레드에서 일어나느냐:

| | VizMotive 현재 방식 | LoadingScreen 패턴 |
|---|---|---|
| copy 처리 스레드 | **메인 스레드** | **워커 스레드** |
| 메인 렌더 루프 | ❌ copy 동안 멈춤 | ✅ 계속 돌아감 |
| copy 방식 | CopyAllocator CPU blocking | CopyAllocator CPU blocking (동일) |

**핵심**: 로딩 완료 체크는 별도 스레드가 담당하고, 씬 반영은 `EVENT_THREAD_SAFE_POINT`
(프레임 경계의 안전한 시점)에서 메인 스레드가 처리한다.
로딩 중에는 2D 로딩 화면(`RenderPath2D`)을 렌더링하면서 getProgress()로 진행률 표시 가능.

### WickedEngine의 해결 방법 2: 리소스 스트리밍 (wiResourceManager.cpp)

텍스처 같은 리소스의 점진적(스트리밍) 로딩:

```cpp
// resourcemanager::Load() — STREAMING 플래그 사용
auto res = resourcemanager::Load("texture.dds", resourcemanager::Flags::STREAMING);
// → 최소 해상도(저해상도 mip)만 즉시 GPU 업로드
// → 나머지는 백그라운드에서 점진적으로 스트리밍

// 매 프레임 호출 — 스트리밍 진행 상태를 업데이트
resourcemanager::UpdateStreamingResources(dt);
```

셰이더에서 렌더링할 때 해당 텍스처의 필요 해상도를 요청:
```cpp
resource.StreamingRequestResolution(required_resolution);
// → UpdateStreamingResources()가 다음 프레임에 더 높은 해상도를 비동기로 업로드
```

**핵심**: 처음에는 저해상도(또는 더미 텍스처)로 보이다가, 백그라운드 스트리밍이 완료되면
점차 고해상도로 대체된다. 프레임 루프를 멈추지 않는다.

### 정리

| 상황 | 권장 방법 |
|---|---|
| 초기 로딩 / 씬 전환 | LoadingScreen: jobsystem 병렬 로딩 + 완료 시 씬 반영 |
| 런타임 텍스처 스트리밍 | `Flags::STREAMING` + `UpdateStreamingResources()` |
| 현재 VizMotive 방식 | 블로킹 CPU wait — **초기 로딩에만 적합** |

이는 DX12 레이어 개선이 아닌, 렌더링 엔진 레벨의 별도 feature 이므로 vizmotive 에 적용 보류