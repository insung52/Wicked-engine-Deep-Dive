# VizMotive 적용: Frame Fence 동기화 단순화

# 개요

WickedEngine의 frame fence 진화 과정(topic_frame_fence_sync.md)을 참고하여,
VizMotive의 fence 구조를 **최종 단순화 형태**(커밋 #21 기반)로 직접 수정한 기록.

---

# Fence의 정확한 의미

> **"동기화(synchronization)" ≠ "종료 때까지 기다림"**
>
> "종료 때까지 기다림"이 동기화를 **보장해 주는 것은 맞지만**,
> 동기화가 "종료 때까지 기다림"을 **의미하는 것은 아니다**.

DX12의 fence는 **GPU 타임라인의 단조 증가(monotonically increasing) 카운터**이다.

fence 기반 메커니즘은 "동기화"라기보다는 더 정확하게:

| 표현 | 의미 |
|------|------|
| **완료 지점(completion point) 추적** | GPU 작업이 어디까지 끝났는지 fence 값으로 확인 |
| **의존성(dependency) 보장 메커니즘** | fence 값을 기준으로 "이 작업이 끝나야 다음 작업 시작 가능" |
| **프레임 재사용을 위한 GPU 완료 확인** | 버퍼를 재사용해도 안전한지 fence 값으로 판단 |

fence 자체는:
- GPU를 멈추는 것이 **아님**
- 전체 파이프라인을 flush하는 것이 **아님**
- 모든 작업을 종료시키는 것이 **아님**

fence가 하는 것:
- 특정 시점까지 완료되었음을 **표시(signal)**
- 그 시점까지 완료될 때까지 **대기(wait)**
- 작업 간 **dependency ordering**을 형성

이 문서에서 fence가 사용되는 세 가지 맥락:

| 맥락 | 구현 | 정확한 설명 |
|------|------|-------------|
| CPU wait | `SetEventOnCompletion` + `WaitForSingleObject` | 프레임 재사용을 위한 GPU 완료 확인 — 버퍼를 재사용해도 안전한지 판단 |
| 큐 간 fence dependency | `queue->Wait(fence, value)` | 큐 간 작업 순서 보장을 위한 fence 기반 dependency — 모든 큐의 현재 프레임 완료 지점에 도달해야 다음 프레임 작업 시작 가능 |
| Signal | `queue->Signal(fence, value)` | GPU 작업 완료 지점 추적 — 이 큐의 여기까지 실행이 끝나면 fence 값을 기록 |

---

# 변경 전 상태 (VizMotive 원본)

## 헤더

```cpp
// GraphicsDevice_DX12.h
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];  // fence만 존재, 따로 값 추적 없음
```

## 초기화 (생성자)

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

## SubmitCommandLists() — Signal

```cpp
// 고정값 1로 signal
hr = queue.queue->Signal(frame_fence[GetBufferIndex()][q].Get(), 1);
```

## SubmitCommandLists() — CPU wait

```cpp
if (FRAMECOUNT >= BUFFERCOUNT && frame_fence[bufferindex][queue]->GetCompletedValue() < 1)
{
    hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(1, NULL);  // 1이 될때까지 대기
}
hr = frame_fence[bufferindex][queue]->Signal(0);  // CPU에서 0으로 리셋
```

## 특징

- 0/1 토글 방식
- 큐 간 fence dependency 없음
- topic_frame_fence_sync.md 기준 **1단계 이전** (가장 원시적인 상태)

---

# 변경 후 상태 (최종 단순화 적용)

## 1. 헤더 변경 (GraphicsDevice_DX12.h:108)

```cpp
// 추가: 버퍼별 fence 값 추적 (monotonically increasing)
uint64_t frame_fence_values[BUFFERCOUNT] = {};

// 기존 유지 (1개 배열)
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];
```

## 2. 초기화 (생성자)

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

## 3. SubmitCommandLists() 변경

전체 흐름:

```
[1] (new) frame_fence_values[buf]++          프레임당 1번만 증가
[2] (edit) Signal 루프                        모든 큐에 같은 값으로 signal
[3] Present
[4] (new) 큐 간 fence dependency 삽입              새로 추가
[5] descriptorheap SignalGPU
[6] FRAMECOUNT++
[7] (edit) CPU wait                           frame_fence_values로 비교, Signal(0) 제거
```

### [1] fence 값 증가 + [2] Signal 루프

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

### [4] 큐 간 작업 순서 보장을 위한 fence 기반 dependency (새로 추가)

> 모든 GPU 큐가 현재 프레임의 **완료 지점(completion point)에 도달해야만**
> 다음 프레임의 GPU 작업이 시작될 수 있도록 하는, fence 기반 dependency 메커니즘.
>
> - 커맨드 리스트 의존성: "큐 내 A 작업이 끝나면 B 시작" (부분적 dependency)
> - **큐 간 fence dependency: "프레임 N의 모든 큐가 완료 지점에 도달해야 프레임 N+1의 어떤 큐도 작업 시작 가능"** (프레임 단위 dependency)

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

이 코드의 의미: 각 큐에 "다른 모든 큐의 현재 프레임 완료 지점에 도달할 때까지 대기"하라는 **dependency를 삽입**.
결과적으로, 모든 큐가 현재 프레임의 완료 지점에 도달해야만 다음 프레임의 어떤 큐도 작업을 시작할 수 없다.

#### + 엔진 내 GPU-GPU fence dependency 두 종류

> | 이름 | 위치 | 용도 |
> |------|------|------|
> | **커맨드 리스트 의존성** (Semaphore) | `commandlist.waits/signals` | 같은 프레임 내에서 커맨드 리스트 A→B 순서 보장 (부분적 dependency) |
> | **큐 간 fence dependency** | `SubmitCommandLists()` 끝 | 프레임 N의 **모든 큐**가 완료 지점에 도달해야 프레임 N+1 시작 가능 (프레임 단위 dependency) |
>
> 여기서 추가한 것은 **큐 간 fence dependency**이다.

#### 문제: 기존 CPU wait는 "같은 버퍼"만 체크한다

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

#### DX12 큐 실행 규칙

- **같은 큐**: 제출 순서 = 실행 순서 (보장됨)
- **다른 큐**: 순서 보장 없음 (완전 독립, 병렬 실행)

#### 큐 간 fence dependency가 없을 때 — 경쟁 조건 발생

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

#### 큐 간 fence dependency 추가 후 — 안전

```
시간 ──────────────────────────────────────────────────────────────────────────▶

Frame 0 (buf 0):                               │ 완료 지점 (모든 큐 완료)
  GRAPHICS:  [████████]                        │
  COMPUTE:   [████████████████████████████████]│
                                               │
Frame 1 (buf 1):                               │                    │ 완료 지점
  GRAPHICS:                                    │[████████████████]  │
  COMPUTE:                                     │       [███████████]│
                                               │                    │
Frame 2 (buf 0):                                                    │
  GRAPHICS:                                                         │[████████████████]
  COMPUTE:                                                          │       [████████████]

모든 큐가 끝나야 다음 프레임의 어떤 큐도 시작할 수 없음 → 경쟁 조건 원천 차단
```

#### CPU wait vs 큐 간 fence dependency 비교

| | CPU wait | 큐 간 fence dependency |
|---|---|---|
| 누가 기다림 | CPU (블로킹) | GPU 큐들 (GPU 내부 대기) |
| 뭘 기다림 | 같은 버퍼의 이전 사용 (2프레임 전) | 현재 프레임의 모든 다른 큐 |
| 목적 | CPU가 GPU보다 2프레임 이상 앞서가지 않게 | 다음 프레임 시작 전 현재 프레임의 모든 GPU 작업 완료 보장 |
| 없으면 | 커맨드 리스트 무한 누적 | 직전 프레임의 느린 큐와 현재 프레임의 빠른 큐가 겹침 |

---

### [7] CPU wait 변경

```cpp
const uint32_t bufferindex = GetBufferIndex();
for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    if (queues[queue].queue == nullptr)
        continue;
    if (FRAMECOUNT >= BUFFERCOUNT
        && frame_fence[bufferindex][queue]->GetCompletedValue() < frame_fence_values[bufferindex])
    {
        // Use event-based wait instead of NULL (spin-wait) to avoid CPU busy-looping:
        //  SetEventOnCompletion registers the fence to signal the event,
        //  then WaitForSingleObject sleeps the thread.
        hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(
            frame_fence_values[bufferindex], frameFenceEvent);
        assert(SUCCEEDED(hr));
        WaitForSingleObject(frameFenceEvent, INFINITE);
    }
    // Signal(0) 제거 — monotonic 증가이므로 리셋 불필요
}
```

---

## SetEventOnCompletion: NULL vs Event 방식

### 변경 이유

`SetEventOnCompletion(value, NULL)`은 내부적으로 **spin-wait(busy loop)** 으로 동작한다.
fence 값에 도달할 때까지 CPU가 계속 polling하며, 스레드가 코어를 100% 점유한 채 대기한다.

CPU wait는 매 프레임 `SubmitCommandLists()`에서 호출되므로, 이 비효율이 매 프레임 누적된다.

### NULL (spin-wait) vs Event (kernel wait)

```
[NULL — spin-wait]
SetEventOnCompletion(value, NULL)  ← 함수가 블로킹, 내부에서 polling

CPU 코어:  [확인][확인][확인][확인][확인][확인][확인][완료!]
            100% 점유, 다른 스레드가 이 코어를 못 씀

[Event — kernel wait]
SetEventOnCompletion(value, event) ← 즉시 리턴, GPU에 "끝나면 event signal해줘" 등록
WaitForSingleObject(event, ...)    ← OS가 스레드를 sleep 상태로 전환

CPU 코어:  [sleep...............................][깨어남!]
            0% 점유, GPU 인터럽트로 깨어남
```

| | NULL (spin-wait) | Event (kernel wait) |
|---|---|---|
| CPU 점유 | 100% (busy loop) | 0% (sleeping) |
| 깨어나는 방식 | 매 루프마다 polling | GPU 하드웨어 인터럽트 → OS가 깨움 |
| 다른 스레드 영향 | 코어 독점 | 코어를 다른 스레드가 사용 가능 |

### 변경 내용

```cpp
// GraphicsDevice_DX12.h — 선언
HANDLE frameFenceEvent = NULL;

// 생성자 — frame_fence 생성 직후, 1번만 생성
frameFenceEvent = CreateEventEx(nullptr, nullptr, 0, EVENT_ALL_ACCESS);

// SubmitCommandLists() CPU wait — NULL을 event로 교체
fence->SetEventOnCompletion(value, frameFenceEvent);  // 즉시 리턴
WaitForSingleObject(frameFenceEvent, INFINITE);        // OS가 스레드를 sleep

// 소멸자 — WaitForGPU() 직후
CloseHandle(frameFenceEvent);
```

Event는 한 번 생성해서 매 프레임 재사용. auto-reset event이므로 signal 후 자동으로 non-signal 상태로 돌아감.

### 다른 NULL 사용처는 수정 불필요

| 위치 | 함수 | 이유 |
|------|------|------|
| 5913줄 | `WaitForGPU()` | 디바이스 소멸/리사이즈 등 드물게 호출. 임시 fence 생성 후 1회 사용. 성능 영향 없음 |
| 7989줄 | Video decode debug | `#if 0`으로 비활성화된 디버그 코드. 실행 안 됨 |

---

## 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| fence 배열 | `frame_fence` 1개 | `frame_fence` 1개 (동일) |
| 값 추적 | 없음 (고정값 1) | `frame_fence_values[BUFFERCOUNT]` 추가 |
| Signal 값 | 1 (고정) | monotonic 증가 (1, 2, 3, ...) |
| CPU 리셋 | `Signal(0)` | 제거 |
| 큐 간 fence dependency | 없음 | 큐 상호 Wait 추가 (모든 GPU 작업 완료 후 다음 프레임) |
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

## 왜 버퍼별 fence인가 (ping-pong과 per-buffer fence)

### ping-pong (더블 버퍼링)이란

`BUFFERCOUNT = 2`, `GetBufferIndex() = FRAMECOUNT % 2`.
CPU와 GPU가 **동시에 다른 프레임을 처리**하기 위해 리소스 슬롯을 2개 번갈아 사용.

```
Frame 0 → buf 0  (ping)
Frame 1 → buf 1  (pong)
Frame 2 → buf 0  (ping)
Frame 3 → buf 1  (pong)

CPU:  [Frame 0 준비(buf 0)] [Frame 1 준비(buf 1)] [Frame 2 준비(buf 0)] ...
GPU:                        [Frame 0 실행(buf 0)] [Frame 1 실행(buf 1)] ...

→ CPU가 buf 1을 준비하는 동안 GPU는 buf 0을 실행 중 — 서로 다른 버퍼이므로 충돌 없음
→ buf 0을 다시 쓰려면(Frame 2) GPU가 Frame 0을 끝낼 때까지 대기 — 이것이 CPU wait
```

### 질문: 왜 `frame_fence[BUFFERCOUNT][QUEUE_COUNT]`인가?

`frame_fence[QUEUE_COUNT]` (큐당 단일 fence)로는 안 되는가?

#### 변경 전 (0/1 토글) — per-buffer 필수

```cpp
// submit 시
queue->Signal(frame_fence[buf][q].Get(), 1);   // fence = 1

// CPU wait 시
if (fence->GetCompletedValue() < 1) ...         // 1이 될 때까지 대기
fence->Signal(0);                                // 리셋!
```

단일 fence라면:

```
Frame 0 (buf 0): Signal(fence, 1)   → fence = 1
Frame 1 (buf 1): Signal(fence, 1)   → ??? 이미 1인데 또 1
                  CPU wait에서 GetCompletedValue() < 1 → false → 대기 안 함!
                  Signal(0)          → fence = 0 → Frame 0 완료 확인이 리셋됨!
```

0/1 토글 방식에서는 버퍼별로 독립적인 fence가 없으면 **리셋이 다른 프레임의 완료 정보를 날림**.

#### 변경 후 (monotonic 증가) — 기술적으로 단일 fence 가능

```cpp
// monotonic 값이므로 리셋 없음, 값이 계속 증가
// 큐당 단일 fence를 쓴다면:
Frame 0: Signal(fence[q], 1)
Frame 1: Signal(fence[q], 2)
Frame 2: Signal(fence[q], 3)  // CPU wait: GetCompletedValue() < 1? → Frame 0 완료 확인 가능
```

monotonic 증가 방식에서는 리셋이 없으므로, 이론적으로 큐당 fence 1개로도 동작한다.

#### 단일 fence로 바꾸면 어떻게 되는가

```cpp
// 단일 fence 구조 (가상)
ComPtr<ID3D12Fence> frame_fence[QUEUE_COUNT];       // 버퍼 차원 없음
uint64_t global_fence_value = 0;                     // 전역 카운터 1개
uint64_t buffer_signal_values[BUFFERCOUNT] = {};     // 각 버퍼가 마지막으로 Signal한 값 추적, 관리할 변수가 증가

// Signal
global_fence_value++;
buffer_signal_values[GetBufferIndex()] = global_fence_value;
queue->Signal(frame_fence[q].Get(), global_fence_value);

// CPU wait — buf 재사용 시
if (fence->GetCompletedValue() < buffer_signal_values[bufferindex]) ...
//                                 ↑ 결국 per-buffer 값 추적이 필요함!
```

기술적으로 동작한다. fence 객체 수는 줄어든다 (`BUFFERCOUNT * QUEUE_COUNT` → `QUEUE_COUNT`).
하지만 **per-buffer signal 값 추적은 여전히 필요**하고, 오히려 관리할 변수가 늘어난다. (uint64_t buffer_signal_values[BUFFERCOUNT])

#### per-buffer를 유지한 이유

1. **0/1 토글 시절에는 필수**: 리셋이 다른 프레임의 완료 정보를 날리므로 fence 분리가 필수였음. monotonic으로 전환할 때 fence 배열 구조까지 바꿀 이유가 없었음
2. **fence 자체가 완료 지점 추적 단위**: `frame_fence[buf][q]`는 "이 버퍼 슬롯의 이 큐의 완료 지점"을 그 자체로 표현. 단일 fence에서는 별도의 `buffer_signal_values` 배열로 간접 추적해야 함
3. **값 관리의 단순함**: per-buffer면 `frame_fence_values[buf]`만으로 Signal과 Wait 양쪽에서 같은 값 사용. 단일 fence면 "이번 프레임에 Signal한 값"과 "이 버퍼의 마지막 Signal 값"을 별도 관리

```cpp
// per-buffer — fence와 값이 1:1 대응, 직관적
queue->Signal(frame_fence[buf][q].Get(), frame_fence_values[buf]);
queue->Wait(frame_fence[buf][q2].Get(), frame_fence_values[buf]);
fence->GetCompletedValue() < frame_fence_values[buf];

// 단일 fence — 값 관리가 간접적
queue->Signal(frame_fence[q].Get(), global_fence_value);
buffer_signal_values[buf] = global_fence_value;              // 추가 추적 필요
queue->Wait(frame_fence[q].Get(),global_fence_value);
fence->GetCompletedValue() < buffer_signal_values[buf];
```

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
