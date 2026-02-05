# 커밋 #3: 93ebdaa6 - dx12 separated cpu and gpu fences

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `93ebdaa6` |
| 날짜 | 2025-03-16 |
| 작성자 | Turánszki János |
| 카테고리 | 안정성 (Stability) |
| 우선순위 | 높음 |
| 선행 커밋 | `3f5a5cc6` (commit #2) - 이 커밋과 **반드시 함께 적용** 필요 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.h | `frame_fence` → `frame_fence_cpu` + `frame_fence_gpu` 분리 (+2줄, -1줄) |
| wiGraphicsDevice_DX12.cpp | 펜스 생성/Signal/Wait 로직 전면 수정 (+35줄, -11줄) |

---

## 문제 상황

### 증상
commit #2 (`3f5a5cc6`)에서 추가된 **프레임 끝 큐 상호 동기화** 로직과 기존 단일 펜스 구조가 충돌하여 **경쟁 조건(Race Condition)** 발생 가능.

### 원인
단일 `frame_fence`를 CPU 대기와 GPU 큐 간 동기화에 **공용으로 사용**하면서, CPU의 펜스 리셋 타이밍과 GPU의 Wait 타이밍이 겹침.

---

## 배경 지식: DX12 Fence

### Fence란?
**Fence(펜스)**는 GPU와 CPU, 또는 GPU 큐들 간의 **동기화 지점**을 나타내는 객체입니다.

```
펜스 = "이 작업까지 완료되면 깃발을 세워줘"

GPU: [작업1] [작업2] [작업3] ──Signal(fence, 100)──▶ 🚩 (fence 값 = 100)

CPU나 다른 큐: Wait(fence, 100) → 🚩가 세워질 때까지 대기
```

### Fence의 두 가지 용도

**1. CPU 대기 (프레임 동기화)**
```cpp
// CPU가 이전 프레임 완료를 기다림
fence->SetEventOnCompletion(1, event);  // fence가 1이 되면 깨워줘
WaitForSingleObject(event, INFINITE);    // 대기
fence->Signal(0);                         // 다음 프레임용으로 리셋
```

**2. GPU-GPU 동기화 (큐 간 동기화)**
```cpp
// 한 큐의 작업 완료를 다른 큐가 기다림
queue_graphics->Signal(fence, 1);        // GRAPHICS 작업 완료 신호
queue_compute->Wait(fence, 1);           // COMPUTE가 GRAPHICS 완료 대기
```

---

## 문제의 시나리오: 단일 펜스의 경쟁 조건

### 단일 펜스 구조 (commit #2 적용 전)

```cpp
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];  // 하나의 펜스
```

commit #2 적용 전에는 GPU-GPU Wait가 없어서 문제 없었음:

```
Frame N:
  GPU Signal(fence, 1)
  CPU SetEventOnCompletion(1) → 완료
  CPU Signal(fence, 0)  ← 리셋

Frame N+1:
  GPU Signal(fence, 1)  ← 새 프레임
  ...반복...
```

### commit #2 적용 후 발생하는 문제

commit #2에서 **프레임 끝 큐 상호 동기화**가 추가됨:

```cpp
// commit #2에서 추가된 코드
for (queue1 in all_queues) {
    for (queue2 in all_queues) {
        queue1->Wait(frame_fence[buffer][queue2], 1);  // GPU-GPU Wait
    }
}
```

이제 **경쟁 조건** 발생:

```
시간선 ────────────────────────────────────────────────────────────────────▶

Frame N 종료:
  ① GPU: Signal(fence, 1)           ← 프레임 완료
  ② CPU: SetEventOnCompletion(1)    ← CPU 대기 완료
  ③ CPU: Signal(fence, 0)           ← 리셋 ⚠️

Frame N+1 시작:
  ④ GPU Queue1: Wait(fence, 1)      ← 이미 0으로 리셋됨! 💥
     → 영원히 대기하거나 타이밍에 따라 잘못 통과
```

### 은행 창구 비유로 이해하기

**상황**: 번호표 시스템이 있는 은행

**등장인물**:
- **고객 (CPU)**: 자기 순서가 되면 업무 처리 후 번호표 리셋
- **은행원들 (GPU 큐들)**: 서로의 번호표를 확인하며 작업 순서 조정

**기존 (commit #2 전)**:
- 고객만 번호표 확인 → 리셋해도 은행원에게 영향 없음

**commit #2 후**:
```
문제의 시나리오:

1. 은행원 A가 번호표 "100번" 발행
2. 고객이 "100번 나왔네" 확인 후 번호표를 "0번"으로 리셋
3. 은행원 B가 "100번 나올 때까지 기다릴게" → 영원히 대기! 💀

은행원 B 입장: "100번 기다리는데 갑자기 0번이 됐네?"
```

---

## 해결: CPU/GPU 펜스 분리

### 핵심 아이디어

**번호표를 두 종류로 분리**:
1. **고객용 번호표 (CPU fence)**: 고객만 확인하고 리셋
2. **은행원용 번호표 (GPU fence)**: 은행원들끼리만 사용, **절대 리셋 안함**

```cpp
// wiGraphicsDevice_DX12.h
ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];  // CPU 전용
ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];  // GPU 전용
```

### Signal 값 전략

| 펜스 | Signal 값 | 대기 값 | 리셋 |
|------|-----------|---------|------|
| `frame_fence_cpu` | `1` (고정) | `1` | `Signal(0)`으로 리셋 |
| `frame_fence_gpu` | `FRAMECOUNT` (증가) | `FRAMECOUNT` | **리셋 안함** (단조 증가) |

### 왜 GPU 펜스는 단조 증가인가?

**리셋 없이 계속 증가**하면 경쟁 조건이 원천 차단됨:

```
Frame 100 (FRAMECOUNT=100):
  GPU Signal(fence_gpu, 100)

Frame 101:
  GPU Queue1: Wait(fence_gpu, 100)  ← fence_gpu는 이미 100
  → 즉시 통과! (100 >= 100)

  GPU Signal(fence_gpu, 101)        ← 단조 증가

Frame 102:
  GPU Queue1: Wait(fence_gpu, 101)  ← fence_gpu는 이미 101
  → 즉시 통과!
```

**핵심**: 펜스 값이 **절대 감소하지 않음** → 과거 프레임 대기는 항상 즉시 통과

### 분리 후 동작 흐름

```
Frame N (FRAMECOUNT=100):
  ① GPU: Signal(fence_cpu, 1)       ← CPU 동기화용
  ② GPU: Signal(fence_gpu, 100)     ← GPU-GPU 동기화용
  ③ GPU Queues: Wait(fence_gpu, 100) ← GPU 펜스로 상호 대기
  ④ CPU: SetEventOnCompletion(fence_cpu, 1) → 완료
  ⑤ CPU: Signal(fence_cpu, 0)       ← CPU 펜스만 리셋

Frame N+1 (FRAMECOUNT=101):
  ① GPU Queues: Wait(fence_gpu, 100) ← 이전 프레임 값, 즉시 통과!
  ② GPU: Signal(fence_cpu, 1)
  ③ GPU: Signal(fence_gpu, 101)     ← 단조 증가
  ...
```

---

## 코드 변경 상세

### 1. 헤더 변경

```cpp
// 변경 전 (wiGraphicsDevice_DX12.h)
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];

// 변경 후
ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];  // CPU 대기용
ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];  // GPU-GPU 동기화용
```

### 2. 펜스 생성

```cpp
// CPU 펜스 - 초기값 1 (free 상태에서 시작)
hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence_cpu[buffer][queue]));
dx12_check(frame_fence_cpu[buffer][queue]->SetName(L"frame_fence_cpu[QUEUE_GRAPHICS]"));

// GPU 펜스 - 초기값 0 (FRAMECOUNT로 증가)
hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence_gpu[buffer][queue]));
dx12_check(frame_fence_gpu[buffer][queue]->SetName(L"frame_fence_gpu[QUEUE_GRAPHICS]"));
```

### 3. Signal 분리 (SubmitCommandLists 내)

```cpp
// CPU용: 고정값 1
queue.queue->Signal(frame_fence_cpu[GetBufferIndex()][q].Get(), 1);

// GPU용: FRAMECOUNT (단조 증가)
queue.queue->Signal(frame_fence_gpu[GetBufferIndex()][q].Get(), FRAMECOUNT);
```

### 4. GPU-GPU Wait (commit #2에서 추가된 부분 수정)

```cpp
// 변경 전 (commit #2)
ID3D12Fence* fence = frame_fence[GetBufferIndex()][queue2].Get();
queues[queue1].queue->Wait(fence, 1);

// 변경 후 (commit #3)
ID3D12Fence* fence = frame_fence_gpu[GetBufferIndex()][queue2].Get();
queues[queue1].queue->Wait(fence, FRAMECOUNT);
```

### 5. CPU 프레임 대기 (기존 로직 유지, 펜스만 변경)

```cpp
ID3D12Fence* fence = frame_fence_cpu[bufferindex][queue].Get();

if (fence->GetCompletedValue() < 1) {
    dx12_check(fence->SetEventOnCompletion(1, nullptr));  // CPU 대기
}

dx12_check(fence->Signal(0));  // CPU 펜스만 리셋 (GPU 펜스는 리셋 안함)
```

---

## 타임라인 비교

### 변경 전 (단일 펜스) - 경쟁 조건 발생

```
시간선 ──────────────────────────────────────────────────────────▶

Frame N:
  GPU Signal(fence, 1) ─────┐
                            │
  CPU Wait(fence, 1) ◀──────┘
  CPU Signal(fence, 0) ──────────┐  ← 리셋
                                 │
Frame N+1:                       │
  GPU Wait(fence, 1) ◀───────────┘  ← 💥 이미 0!
```

### 변경 후 (분리 펜스) - 안전

```
시간선 ──────────────────────────────────────────────────────────▶

Frame N (FRAMECOUNT=100):
  GPU Signal(fence_cpu, 1) ─────┐
  GPU Signal(fence_gpu, 100) ───────────────┐
                                │            │
  GPU Wait(fence_gpu, 100) ◀────│────────────┤  ← GPU 펜스로 대기
  CPU Wait(fence_cpu, 1) ◀──────┘            │
  CPU Signal(fence_cpu, 0) ← CPU만 리셋      │
                                             │
Frame N+1 (FRAMECOUNT=101):                  │
  GPU Wait(fence_gpu, 100) ◀─────────────────┘  ← ✅ 100 >= 100, 즉시 통과!
  GPU Signal(fence_gpu, 101)  ← 단조 증가
```

---

## VizMotive 적용

### 적용 일자
2025-01-26

### 변경 파일
- `GraphicsDevice_DX12.h`: 펜스 배열 분리
- `GraphicsDevice_DX12.cpp`: 펜스 생성/Signal/Wait 로직 수정

### 적용 코드

**1. 헤더 변경 (Line 108-109)**
```cpp
// 변경 전
Microsoft::WRL::ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];

// 변경 후
Microsoft::WRL::ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];
Microsoft::WRL::ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];
```

**2. 펜스 생성 (Lines 2653-2679)**
```cpp
// CPU fence
hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence_cpu[buffer][queue]));
std::wstring fencename_cpu = L"frame_fence_cpu[" + std::to_wstring(buffer) + L"][" + std::to_wstring(queue) + L"]";
dx12_check(frame_fence_cpu[buffer][queue]->SetName(fencename_cpu.c_str()));
dx12_check(frame_fence_cpu[buffer][queue]->Signal(1)); // Initialize to free state

// GPU fence
hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(frame_fence_gpu[buffer][queue]));
std::wstring fencename_gpu = L"frame_fence_gpu[" + std::to_wstring(buffer) + L"][" + std::to_wstring(queue) + L"]";
dx12_check(frame_fence_gpu[buffer][queue]->SetName(fencename_gpu.c_str()));
```

**3. Signal 분리 (Lines 5561-5564)**
```cpp
hr = queue.queue->Signal(frame_fence_cpu[GetBufferIndex()][q].Get(), 1);
hr = queue.queue->Signal(frame_fence_gpu[GetBufferIndex()][q].Get(), FRAMECOUNT);
```

**4. GPU Wait - frame_fence_gpu 사용 (Line 5580)**
```cpp
hr = queue.queue->Wait(frame_fence_gpu[GetBufferIndex()][q2].Get(), FRAMECOUNT);
```

**5. CPU Wait - frame_fence_cpu 사용 (Lines 5642-5652)**
```cpp
ID3D12Fence* fence = frame_fence_cpu[bufferindex][queue].Get();
if (fence->GetCompletedValue() < 1)
{
    dx12_check(fence->SetEventOnCompletion(1, nullptr));
}
dx12_check(fence->Signal(0));  // Wait 후 Signal(0)으로 in-use 마킹
```

---

## 적용 시 주의사항

### 필수: commit #2와 함께 적용

```
적용 순서:
1. commit #2 (3f5a5cc6) - 큐 동기화 추가
   └── 이 시점에서 단일 펜스 사용 시 경쟁 조건 발생 가능 ⚠️

2. commit #3 (93ebdaa6) - 펜스 분리
   └── 경쟁 조건 해결 ✅
```

**주의**:
- commit #2만 적용하고 commit #3을 적용하지 않으면 **경쟁 조건 위험 증가**
- 둘 다 적용하거나, 둘 다 적용하지 않는 것이 안전

### 펜스 오버플로우 고려

`FRAMECOUNT`는 `uint64_t`로, 초당 60프레임 기준:
- 1년: 60 × 60 × 24 × 365 = 약 18.9억
- 오버플로우까지: 2^64 / (60 × 60 × 24 × 365) ≈ **9,780억 년**

→ 실질적으로 오버플로우 걱정 불필요

---

## 효과

1. **CPU 펜스 리셋이 GPU-GPU Wait에 영향 주지 않음**
2. **GPU 펜스는 단조 증가 (FRAMECOUNT)** → 경쟁 조건 원천 차단
3. **멀티큐 동기화 안정성 확보**
4. **디버깅 용이성** - 펜스 이름으로 CPU/GPU 구분 가능

---

## 요약

| 변경 전 | 변경 후 |
|---------|---------|
| 단일 `frame_fence` | `frame_fence_cpu` + `frame_fence_gpu` |
| CPU/GPU 공용 사용 | 용도별 완전 분리 |
| Signal 값 1 (고정) | CPU: 1 (고정), GPU: FRAMECOUNT (증가) |
| CPU 리셋이 GPU에 영향 | CPU 리셋이 GPU와 무관 |
| 경쟁 조건 가능 | 경쟁 조건 원천 차단 |
