# VizMotive 적용: Frame Fence 동기화 단순화

## 개요

WickedEngine의 frame fence 진화 과정(topic_frame_fence_sync.md)을 참고하여,
VizMotive의 fence 구조를 **최종 단순화 형태**(커밋 #21 기반)로 직접 수정한 기록.

---

## 변경 전 상태 (VizMotive 원본)

### 헤더

```cpp
// GraphicsDevice_DX12.h
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];  // fence만 존재, 따로 값 추적 없음
```

### 초기화 (생성자)

```cpp
for (uint32_t buffer = 0; buffer < BUFFERCOUNT; ++buffer)
{
    for (int queue = 0; queue < QUEUE_COUNT; ++queue)
    {
        hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence[buffer][queue]));
        // 초기값 0으로 생성
    }
}
```

### SubmitCommandLists() — Signal

```cpp
// 고정값 1로 signal
hr = queue.queue->Signal(frame_fence[GetBufferIndex()][q].Get(), 1);
```

### SubmitCommandLists() — CPU wait

```cpp
if (FRAMECOUNT >= BUFFERCOUNT && frame_fence[bufferindex][queue]->GetCompletedValue() < 1)
{
    hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(1, NULL);  // 1이 될때까지 대기
}
hr = frame_fence[bufferindex][queue]->Signal(0);  // CPU에서 0으로 리셋
```

### 특징

- 0/1 토글 방식
- GPU-GPU 큐 상호 동기화 없음
- topic_frame_fence_sync.md 기준 **1단계 이전** (가장 원시적인 상태)

---

## 변경 후 상태 (최종 단순화 적용)

### 1. 헤더 변경 (GraphicsDevice_DX12.h:108)

```cpp
// 추가: 버퍼별 fence 값 추적 (monotonically increasing)
uint64_t frame_fence_values[BUFFERCOUNT] = {};

// 기존 유지 (1개 배열)
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];
```

### 2. 초기화 (생성자)

```cpp
for (uint32_t buffer = 0; buffer < BUFFERCOUNT; ++buffer)
{
    frame_fence_values[buffer] = 0;  // 추가
    for (int queue = 0; queue < QUEUE_COUNT; ++queue)
    {
        hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence[buffer][queue]));
    }
}
```

### 3. SubmitCommandLists() 변경

전체 흐름:

```
[1] (new) frame_fence_values[buf]++          프레임당 1번만 증가
[2] (edit) Signal 루프                        모든 큐에 같은 값으로 signal
[3] Present
[4] (new) GPU-GPU 큐 상호 동기화              새로 추가
[5] descriptorheap SignalGPU
[6] FRAMECOUNT++
[7] (edit) CPU wait                           frame_fence_values로 비교, Signal(0) 제거
```

#### [1] fence 값 증가 + [2] Signal 루프

```cpp
// 프레임당 1번만 증가 (루프 밖)
frame_fence_values[GetBufferIndex()]++;

// Mark the completion of queues for this frame:
for (int q = 0; q < QUEUE_COUNT; ++q)
{
    CommandQueue& queue = queues[q];
    if (queue.queue == nullptr)
        continue;

    queue.submit();

    hr = queue.queue->Signal(frame_fence[GetBufferIndex()][q].Get(),
        frame_fence_values[GetBufferIndex()]);  // 기존 signal 1 대신, 증가된 frame_fence_values 값으로 signal
    assert(SUCCEEDED(hr));
}
```

#### [4] GPU-GPU 큐 상호 동기화 (새로 추가)

```cpp
// Sync up every queue to every other queue at the end of the frame:
//  Note: it disables overlapping queues into the next frame
for (int queue1 = 0; queue1 < QUEUE_COUNT; ++queue1)
{
    if (queues[queue1].queue == nullptr)
        continue;
    for (int queue2 = 0; queue2 < QUEUE_COUNT; ++queue2)
    {
        if (queue1 == queue2)
            continue;
        if (queues[queue2].queue == nullptr)
            continue;
        ID3D12Fence* fence = frame_fence[GetBufferIndex()][queue2].Get();
        queues[queue1].queue->Wait(fence, frame_fence_values[GetBufferIndex()]);
    }
}
```

#### + 왜 "프레임 경계 큐 간 동기화"가 필요한가

> **용어 정리**
>
> 이 엔진에는 GPU-GPU 동기화가 두 종류 있다:
>
> | 이름 | 위치 | 용도 |
> |------|------|------|
> | **커맨드 리스트 의존성** (Semaphore) | `commandlist.waits/signals` | 같은 프레임 내에서 커맨드 리스트 A→B 실행 순서 보장 |
> | **프레임 경계 큐 간 동기화** | `SubmitCommandLists()` 끝 | 프레임 N의 **모든 큐**가 끝나야 프레임 N+1의 어떤 큐도 시작 가능 |
>
> 여기서 추가한 것은 **프레임 경계 큐 간 동기화**이다.

##### 문제: CPU wait는 "같은 버퍼"만 체크한다

CPU wait 코드를 보면:

```cpp
// FRAMECOUNT++ 직후 실행
const uint32_t bufferindex = GetBufferIndex();  // FRAMECOUNT % 2
//                                                 ↑ 현재 프레임이 사용할 버퍼만 체크!

for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    if (queues[queue].queue == nullptr)
        continue;

    // bufferindex의 fence만 확인 — 다른 버퍼(= 직전 프레임)는 안 봄!
    if (FRAMECOUNT >= BUFFERCOUNT
        && frame_fence[bufferindex][queue]->GetCompletedValue() < frame_fence_values[bufferindex])
    {
        // CPU가 여기서 블로킹: 이 버퍼의 이전 사용(2프레임 전)이 끝날 때까지 대기
        hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(
            frame_fence_values[bufferindex], NULL);
    }
}
```

BUFFERCOUNT=2이므로, Frame 1 시작 시 buf 1만 확인하고 **buf 0(= Frame 0)은 안 봄**.
Frame 0의 GPU 작업이 아직 진행 중이어도 CPU는 Frame 1을 제출해버림.

##### DX12 큐 실행 규칙

- **같은 큐**: 제출 순서 = 실행 순서 (보장됨)
- **다른 큐**: 순서 보장 없음 (완전 독립, 병렬 실행)

##### 프레임 경계 큐 간 동기화가 없을 때 — 경쟁 조건 발생

```
시간 ──────────────────────────────────────────────────────────────────────────▶

Frame 0 (buf 0):  CPU 제출 → GPU 실행 시작
  GRAPHICS:  [████████]                           ← 빨리 끝남
  COMPUTE:   [███████████████████████]   ← 오래 걸림 (아직 실행 중!)

Frame 1 (buf 1):  CPU는 buf 1만 체크 → 비어있으니 즉시 제출!
  GRAPHICS:                  [███████████]
  COMPUTE:                            [██████████████]
                              ↑
                    Frame 0의 COMPUTE가 아직 GPU에서 실행 중인데
                    Frame 1의 GRAPHICS가 같은 리소스에 접근할 경우 → 경쟁 조건!

Frame 2 (buf 0):  CPU가 buf 0 체크 → Frame 0 완료 대기 후 제출
  GRAPHICS:                               [██████████████████]
  COMPUTE:                                             [███████████]
                                            ↑
                                         Frame 1의 COMPUTE가 아직 실행 중인데
                                         Frame 2의 GRAPHICS가 시작 → 또 경쟁 조건!

Frame 3 (buf 1):  CPU가 buf 1 체크 → Frame 1 완료 대기 후 제출
  GRAPHICS:                                                   [████████]
  COMPUTE:                                                          [████████]

문제: 매 프레임마다 "직전 프레임의 느린 큐"와 "현재 프레임의 빠른 큐"가 겹칠 수 있음
```

##### 프레임 경계 큐 간 동기화 추가 후 — 안전

```
시간 ──────────────────────────────────────────────────────────────────────────▶

Frame 0 (buf 0):                               │ 동기화 지점 (모든 큐 완료)
  GRAPHICS:  [████████]                        │
  COMPUTE:   [████████████████████████████████]│
                                               │
Frame 1 (buf 1):                               │                    │ 동기화 지점
  GRAPHICS:                                    │[████████████████]  │
  COMPUTE:                                     │       [███████████]│
                                               │                    │
Frame 2 (buf 0):                                                    │
  GRAPHICS:                                                         │[████████████████]
  COMPUTE:                                                          │       [████████████]

모든 큐가 끝나야 다음 프레임의 어떤 큐도 시작할 수 없음 → 경쟁 조건 원천 차단
```

##### CPU wait vs 프레임 경계 큐 간 동기화 비교

| | CPU wait | 프레임 경계 큐 간 동기화 |
|---|---|---|
| 누가 기다림 | CPU (블로킹) | GPU 큐들 (GPU 내부 대기) |
| 뭘 기다림 | 같은 버퍼의 이전 사용 (2프레임 전) | 현재 프레임의 모든 다른 큐 |
| 목적 | CPU가 GPU보다 2프레임 이상 앞서가지 않게 | 프레임 간 큐 오버랩 방지 |
| 없으면 | 커맨드 리스트 무한 누적 | 직전 프레임의 느린 큐와 현재 프레임의 빠른 큐가 겹침 |

---

#### [7] CPU wait 변경

```cpp
const uint32_t bufferindex = GetBufferIndex();
for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    if (queues[queue].queue == nullptr)
        continue;
    if (FRAMECOUNT >= BUFFERCOUNT
        && frame_fence[bufferindex][queue]->GetCompletedValue() < frame_fence_values[bufferindex])
    {
        hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(
            frame_fence_values[bufferindex], NULL);
        assert(SUCCEEDED(hr));
    }
    // Signal(0) 제거 — monotonic 증가이므로 리셋 불필요
}
```

---

## 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| fence 배열 | `frame_fence` 1개 | `frame_fence` 1개 (동일) |
| 값 추적 | 없음 (고정값 1) | `frame_fence_values[BUFFERCOUNT]` 추가 |
| Signal 값 | 1 (고정) | monotonic 증가 (1, 2, 3, ...) |
| CPU 리셋 | `Signal(0)` | 제거 |
| GPU-GPU 동기화 | 없음 | 큐 상호 Wait 추가 |
| race condition | CPU 리셋 시 가능 | 원천 차단 (리셋 없음) |

---

## 왜 중간 단계(CPU/GPU fence 분리)를 건너뛰었나

topic_frame_fence_sync.md에서는 4단계 진화 과정을 기록했지만,
실제 적용은 **최종 형태(커밋 #21)로 직접 적용**함.

이유:
1. 중간 단계(frame_fence_cpu/gpu 2개 배열)는 **복잡성만 증가**
2. 최종 형태가 더 단순하고 안전 (monotonic 값 = 리셋 불필요 = race condition 없음)
3. 중간 단계의 문제점을 이해한 상태에서 최종 형태를 적용하므로, 왜 이 구조인지 납득 가능

---



## 왜 frame_fence_values가 필요한가

FRAMECOUNT를 직접 Signal 값으로 쓰면 문제 발생:

```cpp
// fence 초기값: 0
device->CreateFence(0, ...);  // fence.value = 0

// Frame 0: Signal(fence, FRAMECOUNT) → Signal(fence, 0)
// fence.value = 0 ← 초기값이랑 똑같음!

// Frame 2: buf 0 재사용, Frame 0 완료 확인
if (fence->GetCompletedValue() < 0)  // 0 < 0? → false!
    // GPU가 아직 작업 중이어도 여기 안 들어옴!
```

`frame_fence_values`는 per-buffer로 1부터 증가하므로 초기값 0과 겹치지 않음.

---

## 배경 지식 참고

### D3D12_FENCE_FLAG_NONE
- 펜스를 만들 때 옵션을 지정하는 플래그
- NONE은 특별한 기능 없이 기본 펜스를 생성
- 다른 값: D3D12_FENCE_FLAG_SHARED (프로세스 간 공유)

### PPV_ARGS(...)
- Direct3D12 헤더(d3d12.h)에 정의된 매크로
- COM 객체 생성 함수는 마지막 인자로 void** 타입의 포인터를 받음
- PPV_ARGS(ptr)는 `__uuidof(**ptr)`과 `reinterpret_cast<void**>(ptr)`를 자동으로 넣어줌
