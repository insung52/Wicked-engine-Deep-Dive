# 커밋 #21: 3899e472 - Mac OS support (DX12 관련 변경)

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `3899e472` |
| 날짜 | 2026-01-11 |
| 작성자 | Turánszki János |
| 카테고리 | 플랫폼 / 리팩토링 |
| 우선순위 | 낮음 (DX12 부분) |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| 253개 파일 | Mac OS + Metal API 지원 (158,000+ 줄 추가) |
| wiGraphicsDevice_DX12.cpp/h | Fence 단순화, nullptr 체크 추가 |

**참고**: 이 문서는 DX12 관련 변경만 다룹니다. Metal API 관련은 별도 분석 필요.

---

## DX12 변경 1: Fence 단순화

### 배경 지식: GPU Fence

```
┌─────────────────────────────────────────────────────────────────┐
│              DX12 Fence 개념                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Fence: CPU와 GPU 간 동기화 도구                                │
│                                                                 │
│  CPU                              GPU                           │
│   │                                │                            │
│   │ Signal(fence, 1) ─────────────▶│                            │
│   │                                │ (작업 완료 후)             │
│   │                                │ fence.value = 1            │
│   │                                │                            │
│   │ WaitForValue(fence, 1) ◀───────│                            │
│   │ (value >= 1 될 때까지 대기)    │                            │
│   │                                │                            │
│                                                                 │
│  용도:                                                          │
│  - Frame 동기화 (GPU가 프레임 완료할 때까지 대기)               │
│  - 리소스 사용 완료 확인 (삭제 전 GPU 사용 완료 확인)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기존 구현: 두 개의 Fence 배열

```cpp
// 변경 전 - 복잡한 구조
struct GraphicsDevice_DX12
{
    // CPU 측 fence (프레임 사용 중 여부)
    ComPtr<ID3D12Fence> frame_fence_cpu[BUFFERCOUNT][QUEUE_COUNT];
    // GPU 측 fence (GPU 작업 완료 여부)
    ComPtr<ID3D12Fence> frame_fence_gpu[BUFFERCOUNT][QUEUE_COUNT];

    // 사용 방식:
    // - Signal(frame_fence_cpu, 0) = 사용 중
    // - Signal(frame_fence_cpu, 1) = 사용 가능
    // - Signal(frame_fence_gpu, FRAMECOUNT) = GPU 작업 완료
};
```

### 새로운 구현: 단일 Fence + 값 추적

```cpp
// 변경 후 - 단순화된 구조
struct GraphicsDevice_DX12
{
    // 프레임별 fence 값 (monotonically increasing)
    uint64_t frame_fence_values[BUFFERCOUNT] = {};

    // 단일 fence 배열
    ComPtr<ID3D12Fence> frame_fence[BUFFERCOUNT][QUEUE_COUNT];

    // 사용 방식:
    // - 프레임 완료 시 Signal(fence, ++frame_fence_values[buffer])
    // - 대기 시 WaitForValue(fence, frame_fence_values[buffer])
};
```

### 비교 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                      변경 전 (복잡)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Buffer 0:                                                      │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │ frame_fence_cpu │  │ frame_fence_gpu │                       │
│  │ value: 0 or 1   │  │ value: N        │                       │
│  └─────────────────┘  └─────────────────┘                       │
│                                                                 │
│  Buffer 1:                                                      │
│  ┌─────────────────┐  ┌─────────────────┐                       │
│  │ frame_fence_cpu │  │ frame_fence_gpu │                       │
│  │ value: 0 or 1   │  │ value: N        │                       │
│  └─────────────────┘  └─────────────────┘                       │
│                                                                 │
│  로직:                                                          │
│  - cpu fence로 프레임 사용 여부 추적 (0/1 토글)                 │
│  - gpu fence로 GPU 작업 완료 추적                               │
│  → 두 fence 관리 필요, 복잡한 동기화 로직                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      변경 후 (단순)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Buffer 0:                                                      │
│  ┌─────────────────┐  frame_fence_values[0] = 5                 │
│  │  frame_fence    │                                            │
│  │  value: 5       │  (마지막으로 signal한 값)                  │
│  └─────────────────┘                                            │
│                                                                 │
│  Buffer 1:                                                      │
│  ┌─────────────────┐  frame_fence_values[1] = 4                 │
│  │  frame_fence    │                                            │
│  │  value: 4       │                                            │
│  └─────────────────┘                                            │
│                                                                 │
│  로직:                                                          │
│  - 프레임 완료 시 값 증가하며 signal                            │
│  - 해당 버퍼 재사용 전 값 도달 대기                             │
│  → 단일 fence로 모든 동기화 처리                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 코드 변경

```cpp
// 초기화
void GraphicsDevice_DX12::Initialize()
{
    for (int buffer = 0; buffer < BUFFERCOUNT; buffer++)
    {
        frame_fence_values[buffer] = 0;

        for (int queue = 0; queue < QUEUE_COUNT; queue++)
        {
            device->CreateFence(0, D3D12_FENCE_FLAG_NONE,
                IID_PPV_ARGS(&frame_fence[buffer][queue]));
        }
    }
}

// 프레임 제출
void GraphicsDevice_DX12::SubmitCommandLists()
{
    // 현재 버퍼의 fence 값 증가
    frame_fence_values[buffer_index]++;

    // GPU에 signal 요청
    queue->Signal(frame_fence[buffer_index][queue_type].Get(),
                  frame_fence_values[buffer_index]);
}

// GPU 대기
void GraphicsDevice_DX12::WaitForGPU()
{
    for (int buffer = 0; buffer < BUFFERCOUNT; buffer++)
    {
        for (int queue = 0; queue < QUEUE_COUNT; queue++)
        {
            auto fence = frame_fence[buffer][queue].Get();
            uint64_t value = frame_fence_values[buffer];

            if (fence->GetCompletedValue() < value)
            {
                fence->SetEventOnCompletion(value, event);
                WaitForSingleObject(event, INFINITE);
            }
        }
    }
}
```

### 장점

```
1. 코드 단순화
   - Fence 객체 수: 2배 → 1배
   - 동기화 로직 단순화

2. 명확한 의미
   - 값이 monotonically increasing
   - "이 값에 도달하면 작업 완료" 명확

3. 디버깅 용이
   - 현재 fence 값으로 진행 상황 추적 가능
```

---

## DX12 변경 2: WriteTopLevelAccelerationStructureInstance nullptr 체크

### 변경 내용

```cpp
// 변경 전 - nullptr 체크 없음
void GraphicsDevice_DX12::WriteTopLevelAccelerationStructureInstance(
    const RaytracingAccelerationStructureDesc::TopLevel::Instance* instance,
    void* dest)
{
    D3D12_RAYTRACING_INSTANCE_DESC tmp = {};
    // instance가 nullptr이면 크래시!
    tmp.AccelerationStructure = to_internal(instance->bottom_level)->gpu_address;
    // ...
}

// 변경 후 - nullptr 체크 추가
void GraphicsDevice_DX12::WriteTopLevelAccelerationStructureInstance(
    const RaytracingAccelerationStructureDesc::TopLevel::Instance* instance,
    void* dest)
{
    D3D12_RAYTRACING_INSTANCE_DESC tmp = {};

    if (instance != nullptr)  // ✅ nullptr 체크
    {
        auto internal_state = to_internal(instance->bottom_level);
        tmp.AccelerationStructure = internal_state->gpu_address;
        // ... 나머지 설정 ...
    }
    // instance가 nullptr이면 빈 desc 사용 (안전)

    memcpy(dest, &tmp, sizeof(tmp));
}
```

### 왜 필요한가?

```
Top-Level Acceleration Structure (TLAS):
- 여러 Bottom-Level AS (BLAS)를 참조
- 일부 instance가 비어있을 수 있음 (동적 장면)

nullptr instance:
- 해당 슬롯에 객체 없음을 의미
- 빈 desc로 처리하면 안전하게 무시됨
```

---

## 스킵된 변경: 함수명 변경

```cpp
// WickedEngine 변경
AlignTo() → align()
IsAligned() → is_aligned()

// VizMotive: 스킵 (기존 이름 유지)
// 이유: 코드 전반에 걸친 리네임, 기능적 차이 없음
```

---

## VizMotive 적용 현황

### 부분 적용 ✅

**적용 일자**: 2026-01-29

| 파일 | 변경 내용 |
|------|----------|
| GraphicsDevice_DX12.h | `frame_fence_cpu/gpu` → `frame_fence` + `frame_fence_values` |
| GraphicsDevice_DX12.cpp | Fence 초기화 단순화 |
| GraphicsDevice_DX12.cpp | SubmitCommandLists fence signal/wait 로직 수정 |
| GraphicsDevice_DX12.cpp | WaitForGPU fence wait 로직 수정 |
| GraphicsDevice_DX12.cpp | `WriteTopLevelAccelerationStructureInstance` nullptr 체크 |

### 스킵한 부분

| 변경 | 스킵 이유 |
|------|----------|
| Mac OS / Metal API | VizMotive는 Windows/DX12 전용 |
| `AlignTo` → `align` 리네임 | 기능 동일, 불필요한 전체 리네임 |

---

## 요약

| 항목 | 내용 |
|------|------|
| 커밋 목적 | Mac OS + Metal API 지원 |
| DX12 변경 1 | Fence 단순화 (2개 배열 → 1개 + 값 추적) |
| DX12 변경 2 | RT instance nullptr 체크 추가 |
| VizMotive | ✅ DX12 관련 부분만 적용 |

### 핵심 교훈

> **Fence 동기화 단순화**
>
> 복잡한 다중 fence 구조 대신
> 단일 fence + monotonic 값 추적이 더 명확함.
> - 코드 단순화
> - 디버깅 용이
> - 실수 가능성 감소

> **방어적 프로그래밍**
>
> 외부에서 전달받는 포인터는 nullptr 체크.
> 특히 동적으로 변하는 데이터(RT instance 등)는
> 유효하지 않을 수 있음을 고려.
