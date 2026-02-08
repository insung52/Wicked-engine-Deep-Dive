# Topic: Frame Fence & Sync (프레임 펜스 동기화)

## 개요

프레임 간 동기화와 GPU 큐 간 동기화를 위한 Fence 시스템의 진화 과정.

## 관련 커밋

| 순서 | 커밋 | 날짜 | 핵심 변경 |
|------|------|------|----------|
| 1 | [dx1 #2 `3f5a5cc6`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/02_3f5a5cc6_safety_updates.md) | 2025-03-15 | 큐 상호 동기화 추가, race condition 발생 |
| 2 | [dx1 #3 `93ebdaa6`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/03_93ebdaa6_separated_fences.md) | 2025-03-16 | CPU/GPU 펜스 분리로 race condition 해결 |
| 3 | [dx1 #7 `99c82676`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/07_99c82676_dx12_vulkan_improvements.md) | 2025-03-25 | 워크플로우 명확화 (시작/끝 시점) |
| 4 | [dx2 #21 `3899e472`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/21_3899e472_fence_simplification.md) | 2026-01-11 | 2개 배열 → 1개 + 값 추적으로 단순화 |

---

## 변화 흐름

### 1단계: 문제 발견 (커밋 #2)

**배경**: 멀티 큐(Graphics, Compute, Copy) 환경에서 프레임 간 작업 오버랩으로 인한 경쟁 조건 발생 가능성.

**해결 시도**:
```cpp
// 프레임 끝에서 모든 큐가 서로 기다리도록 함
for (queue1 in all_queues) {
    for (queue2 in all_queues) {
        queue1->Wait(frame_fence[buffer][queue2], 1);
    }
}
```

**새로운 문제 발생**: 단일 펜스를 CPU와 GPU가 공유하면서 race condition 발생

```
Frame N 종료:
  ① GPU: Signal(fence, 1)           ← 프레임 완료
  ② CPU: SetEventOnCompletion(1)    ← CPU 대기 완료
  ③ CPU: Signal(fence, 0)           ← 리셋 ⚠️

Frame N+1 시작:
  ④ GPU Queue1: Wait(fence, 1)      ← 이미 0으로 리셋됨! 💥
```

---

### 2단계: CPU/GPU 펜스 분리 (커밋 #3)

**핵심 아이디어**: 펜스를 용도별로 분리

```cpp
// 변경 전
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];

// 변경 후
ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];  // CPU 대기용
ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];  // GPU-GPU 동기화용
```

**Signal 값 전략**:

| 펜스 | Signal 값 | 대기 값 | 리셋 |
|------|-----------|---------|------|
| `frame_fence_cpu` | `1` (고정) | `1` | Signal(0)으로 리셋 |
| `frame_fence_gpu` | `FRAMECOUNT` (증가) | `FRAMECOUNT` | 리셋 안함 (단조 증가) |

**왜 GPU 펜스는 단조 증가인가?**

```
Frame 100: GPU Signal(fence_gpu, 100)
Frame 101: GPU Wait(fence_gpu, 100)  ← 100 >= 100 → 즉시 통과!
           GPU Signal(fence_gpu, 101)

핵심: 펜스 값이 절대 감소하지 않음 → 과거 프레임 대기는 항상 즉시 통과
```

**결과**: CPU 펜스 리셋이 GPU-GPU Wait에 영향 주지 않음

---

### 3단계: 워크플로우 명확화 (커밋 #7)

**문제**: 펜스 값 0과 1의 의미가 시점에 따라 모호함

**해결**: 시작/끝 시점 명확화

```cpp
// 펜스 초기값: 1 (free 상태)
dx12_check(frame_fence_cpu[buffer][queue]->Signal(1));

// 프레임 시작
dx12_check(fence->Signal(0));  // "버퍼 사용 중"

// GPU 작업 제출 후
queue->Signal(fence, 1);  // "GPU 완료"

// 프레임 끝
if (fence->GetCompletedValue() < 1) {
    fence->SetEventOnCompletion(1, nullptr);  // GPU 완료 대기
}
```

**타임라인**:
```
Frame N:
  Signal(0) ──▶ GPU 제출 ──▶ GPU Signal(1) ──▶ CPU Wait(1)
  │                                                   │
  "시작"                                            "끝"
```

---

### 4단계: 최종 단순화 (커밋 #21)

**문제**: 2개 펜스 배열 관리가 여전히 복잡

**해결**: 단일 펜스 + 값 추적

```cpp
// 변경 전 (복잡)
ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];
ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];

// 변경 후 (단순)
uint64_t frame_fence_values[BUFFERCOUNT] = {};  // 마지막 signal 값
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];
```

**동작 방식**:
```cpp
// 프레임 완료 시 값 증가하며 signal
frame_fence_values[buffer_index]++;
queue->Signal(frame_fence[buffer_index][queue].Get(),
              frame_fence_values[buffer_index]);

// 대기 시
if (fence->GetCompletedValue() < frame_fence_values[buffer]) {
    fence->SetEventOnCompletion(frame_fence_values[buffer], event);
    WaitForSingleObject(event, INFINITE);
}
```

---

## 진화 요약

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Fence 구조 진화                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [커밋 #2 전]     단일 펜스, 0/1 토글                                    │
│       │          └── 문제: 큐 오버랩 시 경쟁 조건                        │
│       ▼                                                                 │
│  [커밋 #2]        큐 상호 동기화 추가                                     │
│       │          └── 문제: 단일 펜스로 CPU/GPU 공유 → race condition     │
│       ▼                                                                 │
│  [커밋 #3]        CPU/GPU 펜스 분리                                      │
│       │          ├── CPU 펜스: 0/1 토글 (리셋 가능)                      │
│       │          └── GPU 펜스: FRAMECOUNT 단조 증가 (리셋 안함)          │
│       ▼                                                                 │
│  [커밋 #7]        워크플로우 명확화                                       │
│       │          ├── 초기값 1 (free)                                     │
│       │          ├── 시작 시 Signal(0)                                   │
│       │          └── 완료 시 Signal(1)                                   │
│       ▼                                                                 │
│  [커밋 #21]       최종 단순화                                             │
│                  └── 1개 배열 + 값 추적 (monotonic)                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 핵심 개념 정리

### DX12 Fence란?

GPU-CPU 또는 GPU 큐 간 동기화 도구:

```
CPU ────Signal(1)───▶ [Fence: value=1] ◀───Wait(1)──── GPU Queue
                            │
                      GetCompletedValue() → 현재 값 조회
```

### 두 가지 동기화 방식

| 방식 | 장점 | 단점 | 적합한 상황 |
|------|------|------|------------|
| GPU Wait | CPU 프리 | 복잡한 의존성 | 프레임 내 빈번한 동기화 |
| CPU Wait | 단순함 | CPU 블로킹 | 초기화, 로딩 시점 |

### Monotonic 값의 장점

- 과거 대기는 항상 즉시 통과 (100 >= 100 → pass)
- 리셋 불필요 → race condition 원천 차단
- 디버깅 시 진행 상황 추적 용이

---

## 교훈

1. **단순함이 최고다**: 복잡한 2개 펜스보다 단순한 1개 + 값 추적이 낫다
2. **Monotonic 값**: 리셋이 필요 없으면 단조 증가가 안전하다
3. **용도 분리**: CPU/GPU가 같은 리소스를 다른 방식으로 사용하면 분리 고려
4. **시작/끝 명확화**: 상태의 의미가 시점에 따라 달라지면 명확히 정의

---

## VizMotive 적용

모든 커밋 적용 완료. 최종 구조:

```cpp
// GraphicsDevice_DX12.h
uint64_t frame_fence_values[BUFFERCOUNT] = {};
ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];
```
