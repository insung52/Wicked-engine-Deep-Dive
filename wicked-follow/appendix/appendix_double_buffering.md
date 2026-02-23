# VizMotive 적용: 더블 버퍼링과 GetBufferIndex()

# 개요

이 엔진에서 `BUFFERCOUNT = 2`이다. 이른바 **더블 버퍼링(ping-pong)** 방식으로,
CPU와 GPU가 서로 다른 버퍼 슬롯을 사용해 파이프라이닝한다.

이 문서는 `GetBufferIndex()` 기반으로 엔진의 더블 버퍼링 구조를 코드 레벨에서 정리한다.

- fence 구조 변경 기록 → [apply_frame_fence_sync.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/apply_frame_fence_sync.md)

---

# GetBufferIndex() 정의

```cpp
// GBackendDevice.h:132
constexpr uint32_t GetBufferIndex() const { return GetFrameCount() % GetBufferCount(); }
```

| FRAMECOUNT | GetBufferIndex() |
|------------|------------------|
| 0          | 0                |
| 1          | 1                |
| 2          | 0                |
| 3          | 1                |
| ...        | 0, 1 반복        |

### 주의: SwapChain_DX12::GetBufferIndex()는 별개 함수

```cpp
// GraphicsDevice_DX12.cpp:1522
inline uint32_t GetBufferIndex() const
{
    return swapChain->GetCurrentBackBufferIndex();  // swapchain backbuffer index
}
```

`SwapChain_DX12::GetBufferIndex()`는 **swapchain의 현재 백버퍼 인덱스**를 반환한다.
`GraphicsDevice::GetBufferIndex()`와 이름은 같지만 완전히 다른 값이다.
이 문서에서 `GetBufferIndex()`는 모두 `GraphicsDevice::GetBufferIndex()`를 의미한다.

---

# 더블 버퍼링이 필요한 이유

## BUFFERCOUNT=1일 때 — 파이프라이닝 불가

CPU와 GPU가 같은 리소스를 사용하므로, CPU는 GPU 완료까지 대기해야 한다:

```
CPU:  [Frame 0 준비] [대기........] [Frame 1 준비] [대기........] ...
GPU:                 [Frame 0 실행]                [Frame 1 실행]
                      ↑                             ↑
              CPU는 GPU가 끝날 때까지       다시 대기...
              같은 버퍼를 건드릴 수 없음
```

## BUFFERCOUNT=2일 때 — CPU-GPU 파이프라이닝

CPU가 buf 1을 준비하는 동안 GPU가 buf 0을 실행할 수 있다:

```
Frame 0 → buf 0  (ping)
Frame 1 → buf 1  (pong)
Frame 2 → buf 0  (ping)

CPU:  [Frame 0 준비(buf 0)] [Frame 1 준비(buf 1)] [Frame 2 준비(buf 0)] ...
GPU:                        [Frame 0 실행(buf 0)] [Frame 1 실행(buf 1)] ...
                              ↑
                    CPU가 buf 1로 Frame 1을 준비하는 동안
                    GPU는 buf 0으로 Frame 0을 실행
                    → 서로 다른 버퍼이므로 충돌 없음
```

buf 0을 다시 쓰려면(Frame 2), GPU가 Frame 0을 끝낼 때까지 대기해야 한다 — 이것이 CPU wait이다.

## CPU "준비" 작업이란

CPU가 GPU에 일을 시키기 위해 매 프레임 준비하는 작업들:
- 씬 그래프 업데이트 (transform hierarchy)
- Visibility culling (카메라에서 보이는 오브젝트 판별)
- 커맨드 리스트 기록 (draw call, dispatch, resource barrier 등)
- Descriptor/resource binding
- Shader constant 업데이트

## 결과: 1 frame latency

CPU가 항상 GPU보다 1프레임 앞서 진행한다.
CPU가 Frame N+1을 준비하는 동안 GPU는 Frame N을 실행한다.

---

# SubmitCommandLists()에서 GetBufferIndex() 의미 변화

`SubmitCommandLists()` 내에서 `FRAMECOUNT++` 전후로 `GetBufferIndex()`의 의미가 바뀐다.

## 코드 흐름

```
[1] frame_fence_values[GetBufferIndex()]++              (line 5565)
[2] queue->Signal(fence[GetBufferIndex()][q], ...)      (line 5576)
[3] Present
[4] queue->Wait(fence, ...)                             (line 5639) — 큐 간 dependency
[5] descriptorheap SignalGPU
[6] FRAMECOUNT++                                        (line 5647)
[7] CPU wait: fence[GetBufferIndex()]                   (line 5650-5662)
```

## FRAMECOUNT++ 전: "GPU에 제출 중인 현재 프레임(N)의 버퍼"

```cpp
// line 5565: 현재 프레임(buf)의 fence 값 증가
frame_fence_values[GetBufferIndex()]++;

// line 5576-5577: 현재 프레임(buf)의 각 큐에 Signal
hr = queue.queue->Signal(frame_fence[GetBufferIndex()][q].Get(),
    frame_fence_values[GetBufferIndex()]);

// line 5638-5639: 큐 간 dependency — 현재 프레임(buf)의 완료 지점 기준
ID3D12Fence* fence = frame_fence[GetBufferIndex()][queue2].Get();
queues[queue1].queue->Wait(fence, frame_fence_values[GetBufferIndex()]);
```

이 시점에서 `GetBufferIndex()`는 **방금 GPU에 제출한 프레임(N)의 버퍼 슬롯**이다.

## FRAMECOUNT++ (line 5647)

```cpp
// From here, we begin a new frame, this affects GetBufferIndex()!
FRAMECOUNT++;
```

이 시점에서 `GetBufferIndex()`의 반환값이 바뀐다 (0→1 또는 1→0).

## FRAMECOUNT++ 후: "CPU가 다음에 준비할 프레임(N+1)의 버퍼"

```cpp
// line 5650: 다음 프레임이 사용할 버퍼 슬롯
const uint32_t bufferindex = GetBufferIndex();

// line 5655-5661: 이 버퍼 슬롯의 이전 GPU 사용이 끝났는지 확인
if (FRAMECOUNT >= BUFFERCOUNT
    && frame_fence[bufferindex][queue]->GetCompletedValue() < frame_fence_values[bufferindex])
{
    hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(
        frame_fence_values[bufferindex], frameFenceEvent);
    assert(SUCCEEDED(hr));
    WaitForSingleObject(frameFenceEvent, INFINITE);
}
```

이 시점에서 `GetBufferIndex()`는 **CPU가 다음 프레임(N+1)을 위해 준비할 버퍼 슬롯**이다.
이 버퍼가 2프레임 전 GPU 작업(N-1 프레임)에 의해 아직 사용 중이면, CPU는 여기서 블로킹된다.

---

# GetBufferIndex() 사용처 정리

## BeginCommandList — commandlist.reset(GetBufferIndex())

```cpp
// GraphicsDevice_DX12.cpp:5368
commandlist.reset(GetBufferIndex());
```

`BeginCommandList()`는 `FRAMECOUNT++` 이후에 호출되므로, 이 시점의 `GetBufferIndex()` = **CPU가 이번 프레임의 커맨드 리스트를 준비할 버퍼 슬롯**.

`reset()` 내부 (GraphicsDevice_DX12.h:202):

```cpp
void reset(uint32_t bufferindex)
{
    buffer_index = bufferindex;
    // ...
    frame_allocators[buffer_index].reset();  // 이 버퍼 슬롯의 frame allocator 리셋
    // ...
}
```

- `commandAllocators[buffer_index][queue]`: 이 버퍼 슬롯의 command allocator 선택
- `frame_allocators[buffer_index]`: 이 버퍼 슬롯의 GPU linear allocator 리셋

## GetFrameAllocator(cmd)

```cpp
// GraphicsDevice_DX12.h:435
return GetCommandList(cmd).frame_allocators[GetBufferIndex()];
```

현재 프레임의 버퍼 슬롯에 해당하는 GPU linear allocator를 반환한다.

## SubmitCommandLists() — fence 관련

위의 "[SubmitCommandLists()에서 GetBufferIndex() 의미 변화](#submitcommandlists에서-getbufferindex-의미-변화)" 섹션에서 설명.

## AllocationHandler::Update() — 리소스 지연 삭제

**상세 Bindless Resources 개념 및 Deferred Deletion 정리 문서 : [appendix_bindless_resources_and_deferred_deletion.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/appendix/appendix_bindless_resources_and_deferred_deletion.md)**

```cpp
// GraphicsDevice_DX12.h:548
void Update(uint64_t FRAMECOUNT, uint32_t BUFFERCOUNT)
{
    // ...
    while (!destroyer_allocations.empty()
        && destroyer_allocations.front().second + BUFFERCOUNT < FRAMECOUNT)
    {
        destroyer_allocations.pop_front();
    }
    // ... (destroyer_resources, destroyer_pipelines 등 동일 패턴)

    // bindless descriptor 재활용
    while (!destroyer_bindless_res.empty()
        && destroyer_bindless_res.front().second + BUFFERCOUNT < FRAMECOUNT)
    {
        int index = destroyer_bindless_res.front().first;
        destroyer_bindless_res.pop_front();
        free_bindless_res.push_back(index);  // 재활용 풀로 반환
    }
    // destroyer_bindless_sam도 동일
}
```

호출 시점 (GraphicsDevice_DX12.cpp:5667):

```cpp
allocationhandler->Update(FRAMECOUNT, BUFFERCOUNT);
```

- 리소스 삭제 요청 시 현재 `FRAMECOUNT`를 기록
- `BUFFERCOUNT` 프레임이 지난 후 안전하게 해제 (GPU가 더 이상 사용하지 않음이 보장)
- bindless descriptor의 경우, 해제 후 `free_bindless_res`/`free_bindless_sam`으로 반환하여 재활용

`GetBufferIndex()`를 직접 호출하지는 않지만, **BUFFERCOUNT 프레임 지연 삭제**라는 더블 버퍼링의 핵심 전제에 의존한다.

---

# 왜 frame_fence[BUFFERCOUNT][QUEUE_COUNT]인가

fence도 버퍼별로 존재하는 이유.

## 변경 전 (0/1 토글) — per-buffer 필수

```cpp
// submit 시
queue->Signal(frame_fence[buf][q].Get(), 1);   // fence = 1

// CPU wait 시
if (fence->GetCompletedValue() < 1) ...         // 1이 될 때까지 대기
fence->Signal(0);                                // 리셋!
```

큐당 단일 fence라면:

```
Frame 0 (buf 0): Signal(fence, 1)   → fence = 1
Frame 1 (buf 1): Signal(fence, 1)   → ??? 이미 1인데 또 1
                  CPU wait에서 GetCompletedValue() < 1 → false → 대기 안 함!
                  Signal(0)          → fence = 0 → Frame 0 완료 확인이 리셋됨!
```

0/1 토글 방식에서는 버퍼별로 독립적인 fence가 없으면 **리셋이 다른 프레임의 완료 정보를 날림**.

## 변경 후 (monotonic 증가) — 기술적으로 단일 fence 가능

```cpp
// monotonic 값이므로 리셋 없음, 값이 계속 증가
// 큐당 단일 fence를 쓴다면:
Frame 0: Signal(fence[q], 1)
Frame 1: Signal(fence[q], 2)
Frame 2: Signal(fence[q], 3)  // CPU wait: GetCompletedValue() < 1? → Frame 0 완료 확인 가능
```

monotonic 증가 방식에서는 리셋이 없으므로, 이론적으로 큐당 fence 1개로도 동작한다.

## 단일 fence로 바꾸면

```cpp
// 단일 fence 구조 (가상)
ComPtr<ID3D12Fence> frame_fence[QUEUE_COUNT];       // 버퍼 차원 없음
uint64_t global_fence_value = 0;                     // 전역 카운터 1개
uint64_t buffer_signal_values[BUFFERCOUNT] = {};     // 각 버퍼가 마지막으로 Signal한 값 추적

// Signal
global_fence_value++;
buffer_signal_values[GetBufferIndex()] = global_fence_value;
queue->Signal(frame_fence[q].Get(), global_fence_value);

// CPU wait — buf 재사용 시
if (fence->GetCompletedValue() < buffer_signal_values[bufferindex]) ...
//                                 ↑ 결국 per-buffer 값 추적이 필요함!
```

기술적으로 동작한다. fence 객체 수는 줄어든다 (`BUFFERCOUNT * QUEUE_COUNT` → `QUEUE_COUNT`).
하지만 **per-buffer signal 값 추적은 여전히 필요**하고, 오히려 관리할 변수가 늘어난다.

## per-buffer를 유지한 이유

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
queue->Wait(frame_fence[q].Get(), global_fence_value);
fence->GetCompletedValue() < buffer_signal_values[buf];
```

---

# 배경 지식

## D3D12_FENCE_FLAG_NONE
- 펜스를 만들 때 옵션을 지정하는 플래그
- NONE은 특별한 기능 없이 기본 펜스를 생성
- 다른 값: `D3D12_FENCE_FLAG_SHARED` (프로세스 간 공유)

## PPV_ARGS(...)
- Direct3D12 헤더(d3d12.h)에 정의된 매크로
- COM 객체 생성 함수는 마지막 인자로 `void**` 타입의 포인터를 받음
- `PPV_ARGS(ptr)`는 `__uuidof(**ptr)`과 `reinterpret_cast<void**>(ptr)`를 자동으로 넣어줌
