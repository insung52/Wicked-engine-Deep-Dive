# VizMotive 적용: Frame Fence 동기화 단순화

# 개요

WickedEngine의 frame fence 진화 과정(topic_frame_fence_sync.md)을 참고하여,
VizMotive의 fence 구조를 **최종 단순화 형태**(커밋 #21 기반)로 직접 수정한 기록.

- 더블 버퍼링 개념, `GetBufferIndex()` 코드 분석, 사용처 정리 → [apply_double_buffering.md](apply_double_buffering.md)

---

# Fence 란

> **"동기화(synchronization)" ≠ "종료 때까지 기다림"**
>
> "종료 때까지 기다림"이 동기화를 **보장해 주는 것은 맞지만**,
> 동기화가 "종료 때까지 기다림"을 **의미하는 것은 아니다**.

DX12의 fence는 **GPU 타임라인의 단조 증가(monotonically increasing) 카운터**이다.
이를 통해 작업 간 dependency ordering을 형성한다.

fence가 하는 것:
- 특정 시점까지 완료되었음을 **표시(Signal)**
- 그 시점까지 완료될 때까지 **대기(Wait)**
- 작업 간 **dependency ordering**을 형성

fence 자체는:
- GPU를 멈추는 것이 **아님**
- 전체 파이프라인을 flush하는 것이 **아님**
- 모든 작업을 종료시키는 것이 **아님**

#### 용어

- **큐(queue)**: DX12의 command queue (`ID3D12CommandQueue`). 이 엔진에서는 GRAPHICS, COMPUTE, COPY, VIDEO_DECODE 4종류가 존재.
- **프레임 완료 지점**: 한 프레임에서 각 큐에 제출된 모든 커맨드 리스트의 실행이 끝난 시점. 코드상으로는 `queue.submit()` 이후 `queue->Signal(fence, value)`로 표시(mark)하는 지점.

이 문서에서 fence가 사용되는 세 가지 맥락:

| 맥락 | 구현 | 설명 |
|------|------|------|
| **Signal** | `queue->Signal(fence, value)` | GPU 작업 완료 지점 추적 — 현재 프레임의 모든 커맨드 리스트 실행이 끝나면 fence 값을 기록 |
| **큐 간 fence dependency** | `queue->Wait(fence, value)` | 모든 GPU 큐가 현재 프레임의 완료 지점에 도달해야 다음 프레임 작업 시작 가능 |
| **CPU wait** | `SetEventOnCompletion` + `WaitForSingleObject` | CPU가 frame N 을 submit 후, frame N+1 을 준비하기 위해 frame N-1 의 버퍼 슬롯의 이전 GPU 작업이 완료되었는지 확인 (더블 버퍼링 상세: [apply_double_buffering.md](apply_double_buffering.md)) |

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
// FRAMECOUNT++ 이후
if (FRAMECOUNT >= BUFFERCOUNT && frame_fence[bufferindex][queue]->GetCompletedValue() < 1)
{
    hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(1, NULL);  // 1이 될때까지 대기
}
hr = frame_fence[bufferindex][queue]->Signal(0);  // CPU에서 0으로 리셋
```

## 문제점

- 0/1 토글 방식
- 큐 간 fence dependency 없음
- topic_frame_fence_sync.md 기준 **1단계 이전** (가장 원시적인 상태)

---

# 변경 후 상태 (최종 단순화 적용)

## 1. 헤더 변경 (GraphicsDevice_DX12.h:108-113)

```cpp
// 기존 유지
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];

// 추가: 버퍼별 fence 값 추적 (monotonically increasing)
uint64_t frame_fence_values[BUFFERCOUNT] = {};

// 추가: CPU wait용 이벤트 핸들
HANDLE frameFenceEvent = NULL;
```

## 2. 초기화 변경 (생성자)

```cpp
for (uint32_t buffer = 0; buffer < BUFFERCOUNT; ++buffer)
{
    frame_fence_values[buffer] = 0;  // 추가
    for (int queue = 0; queue < QUEUE_COUNT; ++queue)
    {
        hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence[buffer][queue]));
    }
}

// frame_fence 생성 직후, 1번만 생성 (line 2669)
frameFenceEvent = CreateEventEx(nullptr, nullptr, 0, EVENT_ALL_ACCESS);
```

## 3. SubmitCommandLists() 변경

전체 흐름:

| 단계 | 내용 | 변경 |
|------|------|------|
| [1] | `frame_fence_values[buf]++` — 프레임당 1번만 증가 | new |
| [2] | Signal 루프 — 모든 큐에 같은 값으로 signal | edit |
| [3] | Present | — |
| [4] | 큐 간 fence dependency 삽입 | new |
| [5] | descriptorheap SignalGPU | — |
| [6] | `FRAMECOUNT++` | — |
| [7] | CPU wait — `frame_fence_values`로 비교, `Signal(0)` 제거 | edit |

### [1]+[2] fence 값 증가 + Signal

```cpp
// 프레임당 1번만 증가 (루프 밖, line 5565)
frame_fence_values[GetBufferIndex()]++;

// Mark the completion of queues for this frame:
for (int q = 0; q < QUEUE_COUNT; ++q)
{
    CommandQueue& queue = queues[q];
    if (queue.queue == nullptr)
        continue;

    queue.submit();

    // 기존 signal 1 대신, 증가된 frame_fence_values 값으로 signal (line 5576-5577)
    hr = queue.queue->Signal(frame_fence[GetBufferIndex()][q].Get(),
        frame_fence_values[GetBufferIndex()]);
    assert(SUCCEEDED(hr));
}
```

이 시점의 `GetBufferIndex()` = **현재 프레임의 버퍼 슬롯** (`FRAMECOUNT++` 전).

### [4] 큐 간 fence dependency (새로 추가)

```cpp
// Sync up every queue to every other queue at the end of the frame:
//  Note: it disables overlapping queues into the next frame  (line 5628-5641)
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

각 큐에 "다른 모든 큐의 현재 프레임 완료 지점에 도달할 때까지 대기"하라는 **dependency를 삽입**.
결과: 모든 큐가 현재 프레임의 완료 지점에 도달해야만 다음 프레임의 큐 작업을 시작 가능.

#### 왜 필요한가: CPU wait는 "같은 버퍼"만 체크한다

CPU wait 코드를 보면:

```cpp
// FRAMECOUNT++ 직후 실행
const uint32_t bufferindex = GetBufferIndex();  // FRAMECOUNT % 2
//                                ↑ CPU가 다음에 준비할 버퍼만 체크!

for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    // bufferindex의 fence만 확인 — 다른 버퍼(= 방금전에 submit 한 버퍼)는 안 봄!
    if (frame_fence[bufferindex][queue]->GetCompletedValue() < frame_fence_values[bufferindex])
    { ... }
}
```

BUFFERCOUNT=2이므로, Frame 1 을 사용하기 전에 buf 1만 확인하고 **buf 0(= Frame 0)은 안 봄**.
Frame 0의 GPU 작업이 아직 진행 중이어도 CPU는 Frame 1을 작업 시작 및 제출할 수 있다.

큐 간 fence dependency가 없으면, GPU 내부에서 **의도하지 않은 프레임 간 병렬 처리**가 발생한다:
Frame N의 느린 큐(예: COMPUTE)가 아직 실행 중인데, Frame N+1의 빠른 큐(예: GRAPHICS)가
이미 시작되어, 서로 다른 프레임의 작업이 서로 다른 큐에서 동시에 실행된다.

#### 이 코드가 하는 것

모든 큐가 현재 프레임의 완료 지점에 도달해야만 다음 프레임의 어떤 큐도 시작할 수 없도록,
GPU 내부에서 프레임 경계를 강제한다. (상세 다이어그램은 [문서 하단](#참고-다이어그램) 참고)

### [7] CPU wait 변경

```cpp
// FRAMECOUNT++ 후 실행 (line 5650-5662)
const uint32_t bufferindex = GetBufferIndex();
for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    if (queues[queue].queue == nullptr)
        continue;
    if (FRAMECOUNT >= BUFFERCOUNT
        && frame_fence[bufferindex][queue]->GetCompletedValue() < frame_fence_values[bufferindex])
    {
        // Use event-based wait instead of NULL (spin-wait) to avoid CPU busy-looping:
        hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(
            frame_fence_values[bufferindex], frameFenceEvent);
        assert(SUCCEEDED(hr));
        WaitForSingleObject(frameFenceEvent, INFINITE);
    }
    // Signal(0) 제거 — monotonic 증가이므로 리셋 불필요
}
```

이 시점의 `GetBufferIndex()` = **CPU가 다음에 준비할 버퍼 슬롯** (`FRAMECOUNT++` 후).
이 버퍼 슬롯의 이전 GPU 사용이 끝났는지 확인하고, 끝나지 않았으면 CPU가 블로킹된다.

#### SetEventOnCompletion: NULL → Event 방식으로 변경한 이유

`SetEventOnCompletion(value, NULL)`은 내부적으로 **spin-wait(busy loop)** 으로 동작한다.
fence 값에 도달할 때까지 CPU가 계속 polling하며, 스레드가 코어를 100% 점유한 채 대기한다.
CPU wait는 매 프레임 호출되므로 이 비효율이 매 프레임 누적된다.

```cpp
// GraphicsDevice_DX12.h:111 — 선언
HANDLE frameFenceEvent = NULL;

// 생성자 (line 2669) — 1번만 생성
frameFenceEvent = CreateEventEx(nullptr, nullptr, 0, EVENT_ALL_ACCESS);

// SubmitCommandLists() CPU wait — NULL을 event로 교체
fence->SetEventOnCompletion(value, frameFenceEvent);  // 즉시 리턴
WaitForSingleObject(frameFenceEvent, INFINITE);        // OS가 스레드를 sleep
```

Event는 한 번 생성해서 매 프레임 재사용. auto-reset event이므로 signal 후 자동으로 non-signal 상태로 돌아감.

## 4. 소멸자 추가

```cpp
// WaitForGPU() 직후 (line 3086)
CloseHandle(frameFenceEvent);
```

---

# 변경 전후 비교

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| fence 배열 | `frame_fence` 1개 | `frame_fence` 1개 (동일) |
| 값 추적 | 없음 (고정값 1) | `frame_fence_values[BUFFERCOUNT]` 추가 |
| Signal 값 | 1 (고정) | monotonic 증가 (1, 2, 3, ...) |
| CPU 리셋 | `Signal(0)` | 제거 |
| CPU wait 방식 | `SetEventOnCompletion(1, NULL)` (spin-wait) | Event 기반 (kernel wait) |
| 큐 간 fence dependency | 없음 | 큐 상호 Wait 추가 |
| race condition | CPU 리셋 시 가능 | 원천 차단 (리셋 없음 + 큐 간 dependency) |

---

# 보충 설명

## 왜 frame_fence_values가 필요한가 (FRAMECOUNT 직접 사용 불가)

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

## 왜 중간 단계를 건너뛰었나

topic_frame_fence_sync.md에서는 4단계 진화 과정을 기록했지만,
실제 적용은 **최종 형태(커밋 #21)로 직접 적용**함.

이유:
1. 중간 단계(frame_fence_cpu/gpu 2개 배열)는 **복잡성만 증가**
2. 최종 형태가 더 단순하고 안전 (monotonic 값 = 리셋 불필요 = race condition 없음)
3. 중간 단계의 문제점을 이해한 상태에서 최종 형태를 적용하므로, 왜 이 구조인지 납득 가능

## SetEventOnCompletion NULL: 다른 사용처는 수정 불필요

| 위치 | 함수 | 이유 |
|------|------|------|
| line 5913 | `WaitForGPU()` | 디바이스 소멸/리사이즈 등 드물게 호출. 임시 fence 생성 후 1회 사용. 성능 영향 없음 |
| line 7989 | Video decode debug | `#if 0`으로 비활성화된 디버그 코드. 실행 안 됨 |

---

# 참고 다이어그램

## 큐 간 fence dependency가 없을 때 — 경쟁 조건

DX12 큐 실행 규칙:
- **같은 큐**: 제출 순서 = 실행 순서 (보장됨)
- **다른 큐**: 순서 보장 없음 (완전 독립, 병렬 실행)

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

문제: 매 프레임마다 "직전 프레임의 느린 큐"와 "현재 프레임의 빠른 큐"가 겹칠 수 있음
```

## Worst case (Frame 5/6)

프레임은 월드 시간 순서로 업데이트되어야 한다 (Frame 5 → Frame 6 → Frame 7 ...).
하지만 큐 간 fence dependency가 없으면, GPU 내부에서 프레임 완료 순서가 뒤집힐 수 있다:

```
Frame 5 (buf 1):
  GRAPHICS:  [████]                                        ← 빨리 끝남
  COMPUTE:   [██████████████████████████████████████████]   ← 매우 느림 (아직 실행 중!)

Frame 6 (buf 0):  CPU는 buf 0만 체크 → Frame 4 끝났으니 즉시 제출
  GRAPHICS:            [████████]   ← Frame 5의 COMPUTE보다 먼저 끝남!
  COMPUTE:                                                 [████████]
                                                            ↑ 같은 큐이므로 Frame 5 COMPUTE 끝난 후에야 시작

                         ↑
          Frame 6의 GRAPHICS가 Frame 5의 COMPUTE보다 먼저 완료됨!
```

Frame 6의 GRAPHICS가 Frame 5의 COMPUTE 결과(예: 물리 시뮬레이션, 파티클 계산)를
읽어야 하는 경우, 아직 계산이 끝나지 않은 Frame 5의 중간 상태를 읽게 된다.

## 큐 간 fence dependency 추가 후 — 안전한 실행

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

## CPU wait vs 큐 간 fence dependency 비교

| | CPU wait | 큐 간 fence dependency |
|---|---|---|
| 누가 기다림 | CPU (블로킹) | GPU 큐들 (GPU 내부 대기) |
| 뭘 기다림 | n 프레임 submit 후 n+1 프레임 작업을 위해 n-1 프레임의 버퍼 슬롯의 이전 gpu 사용 대기 | 현재 프레임의 모든 다른 큐 |
| 목적 | CPU가 GPU보다 2프레임 이상 앞서가지 않게 | 다음 프레임 시작 전 현재 프레임의 모든 GPU 작업 완료 보장 |
| 없으면 | 커맨드 리스트 무한 누적 | 직전 프레임의 느린 큐와 현재 프레임의 빠른 큐가 겹침 |

## NULL vs Event 비교

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
