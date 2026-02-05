# 커밋 #2: 3f5a5cc6 - dx12 and vulkan additional safety updates

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `3f5a5cc6be856afccb4db2ac6c6f500ea1d7e321` |
| 날짜 | 2025-03-15 |
| 작성자 | Turánszki János |
| 카테고리 | 안정성 (Stability) |
| 우선순위 | 높음 |

## 변경 파일 요약

| 파일 | 변경량 | 주요 내용 |
|------|--------|----------|
| wiGraphicsDevice_DX12.cpp | +64줄 | 펜스 명명, CopyAllocator 변경, 큐 동기화 |
| wiGraphicsDevice_DX12.h | +1줄 | DependencySemaphore 펜스 명명 |
| wiGraphicsDevice_Vulkan.cpp | +207줄 | Vulkan 동일 변경 (본 문서에서는 생략) |

---

## 배경 지식: HRESULT와 dx12_check

### HRESULT란?

Windows COM/DirectX API에서 사용하는 **32비트 오류 코드**입니다.

```cpp
typedef long HRESULT;
```

#### HRESULT 구조 (32비트)

```
┌─────────┬──────────┬────────────────────────────┐
│ 비트 31 │ 비트 30-16 │        비트 15-0          │
│ Severity│ Facility  │         Code              │
│ (성공/실패)│ (출처)    │       (오류 코드)          │
└─────────┴──────────┴────────────────────────────┘
```

- **비트 31 (Severity)**: 0 = 성공, 1 = 실패
- **Facility**: 어떤 시스템에서 발생했는지 (D3D12, DXGI 등)
- **Code**: 구체적인 오류 번호

#### 주요 HRESULT 값

```cpp
S_OK                     = 0x00000000  // 성공
S_FALSE                  = 0x00000001  // 성공 (하지만 뭔가 다름)
E_FAIL                   = 0x80004005  // 일반 실패
E_INVALIDARG             = 0x80070057  // 잘못된 인자
E_OUTOFMEMORY            = 0x8007000E  // 메모리 부족
DXGI_ERROR_DEVICE_REMOVED = 0x887A0005  // GPU 장치 제거됨
D3D12_ERROR_ADAPTER_NOT_FOUND = 0x887E0001  // 어댑터 없음
```

#### HRESULT 체크 매크로

```cpp
// 성공 여부 확인
SUCCEEDED(hr)  // 비트 31이 0이면 true
FAILED(hr)     // 비트 31이 1이면 true
```

### dx12_check 매크로 분석

WickedEngine에서 정의한 DX12 전용 에러 체크 매크로입니다.

```cpp
// wiGraphicsDevice_DX12.h

#define dx12_assert(cond, fname) { \
    wilog_assert(cond, \
        "DX12 error: %s failed with %s (%s:%d)", \
        fname, \
        wi::helper::GetPlatformErrorString(hr).c_str(), \
        relative_path(__FILE__), \
        __LINE__); \
}

#define dx12_check(call) \
    [&]() { \
        HRESULT hr = call; \
        dx12_assert(SUCCEEDED(hr), extract_function_name(#call).c_str()); \
        return hr; \
    }()
```

#### 동작 방식

```cpp
// 사용 예
dx12_check(device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(fence)));

// 위 코드는 다음과 같이 확장됨:
[&]() {
    HRESULT hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE, PPV_ARGS(fence));
    if (!SUCCEEDED(hr)) {
        // 에러 로그 출력:
        // "DX12 error: CreateFence failed with 0x80004005 (GraphicsDevice_DX12.cpp:1234)"
    }
    return hr;
}()
```

#### 왜 람다를 사용하나?

```cpp
// 1. 지역 변수 hr을 캡처하면서
// 2. 표현식처럼 사용 가능 (반환값 활용)

HRESULT result = dx12_check(device->CreateBuffer(...));  // ✅ 반환값 사용 가능
if (FAILED(result)) { ... }
```

#### dx12_check vs 단순 assert

```cpp
// 단순 assert - 정보 부족
assert(SUCCEEDED(device->CreateFence(...)));
// 실패 시: "Assertion failed: SUCCEEDED(...)"

// dx12_check - 상세 정보 제공
dx12_check(device->CreateFence(...));
// 실패 시: "DX12 error: CreateFence failed with E_OUTOFMEMORY (GraphicsDevice_DX12.cpp:1234)"
```

---

## 변경 1: 모든 Fence에 디버그 이름 설정

### 목적

GPU 디버깅 툴(PIX, NSight, RenderDoc 등)에서 Fence를 식별하기 위함.

### DX12 Fence란?

GPU-CPU 또는 GPU 큐 간 동기화를 위한 객체입니다.

```
CPU ────Signal(1)───▶ [Fence: value=1] ◀───Wait(1)──── GPU Queue
                            │
                      GetCompletedValue() → 현재 값 조회
```

### 추가된 코드

#### 1. Frame Fence 명명

```cpp
for (int buffer = 0; buffer < BUFFERCOUNT; ++buffer)
{
    for (int queue = 0; queue < QUEUE_COUNT; ++queue)
    {
        hr = device->CreateFence(0, D3D12_FENCE_FLAG_NONE,
                                 PPV_ARGS(frame_fence[buffer][queue]));

        // 새로 추가된 부분
        switch (queue)
        {
        case QUEUE_GRAPHICS:
            dx12_check(frame_fence[buffer][queue]->SetName(
                L"frame_fence[QUEUE_GRAPHICS]"));
            break;
        case QUEUE_COMPUTE:
            dx12_check(frame_fence[buffer][queue]->SetName(
                L"frame_fence[QUEUE_COMPUTE]"));
            break;
        case QUEUE_COPY:
            dx12_check(frame_fence[buffer][queue]->SetName(
                L"frame_fence[QUEUE_COPY]"));
            break;
        case QUEUE_VIDEO_DECODE:
            dx12_check(frame_fence[buffer][queue]->SetName(
                L"frame_fence[QUEUE_VIDEO_DECODE]"));
            break;
        }
    }
}
```

#### 2. 기타 Fence 명명

```cpp
// Descriptor Heap 펜스
dx12_check(descriptorheap_res.fence->SetName(
    L"DescriptorHeapGPU[CBV_SRV_UAV]::fence"));
dx12_check(descriptorheap_sam.fence->SetName(
    L"DescriptorHeapGPU[SAMPLER]::fence"));

// CopyAllocator 펜스
dx12_check(cmd.fence->SetName(L"CopyAllocator::fence"));

// Device Removed 감지용 펜스
dx12_check(deviceRemovedFence->SetName(L"deviceRemovedFence"));

// 의존성 세마포어 펜스
dx12_check(dependency.fence.Get()->SetName(L"DependencySemaphore"));
```

### PIX에서의 효과

**명명 전:**
```
Fence 0x00000001234
Fence 0x00000001238
Fence 0x0000000123C
...
```

**명명 후:**
```
frame_fence[QUEUE_GRAPHICS]
frame_fence[QUEUE_COMPUTE]
CopyAllocator::fence
deviceRemovedFence
...
```

---

## 변경 2: CopyAllocator CPU 대기로 변경

### CopyAllocator란?

텍스처, 버퍼 등 리소스를 GPU 메모리로 업로드하는 전용 시스템입니다.

```
[CPU 메모리]                    [GPU 메모리]
     │                              ▲
     ▼                              │
[Upload Buffer] ──CopyAllocator──▶ [Resource]
  (UPLOAD heap)     (COPY Queue)   (DEFAULT heap)
```

### 변경 전: GPU 큐 대기 방식

```cpp
void CopyAllocator::submit(CopyCMD cmd)
{
    // 복사 커맨드 실행
    queue->ExecuteCommandLists(1, commandlists);

    // COPY 큐가 완료되면 Fence 신호
    dx12_check(queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled));

    // 모든 GPU 큐가 복사 완료를 대기
    dx12_check(device->queues[QUEUE_GRAPHICS].queue->Wait(
        cmd.fence.Get(), cmd.fenceValueSignaled));
    dx12_check(device->queues[QUEUE_COMPUTE].queue->Wait(
        cmd.fence.Get(), cmd.fenceValueSignaled));
    dx12_check(device->queues[QUEUE_COPY].queue->Wait(
        cmd.fence.Get(), cmd.fenceValueSignaled));
    // VIDEO_DECODE도 동일...
}
```

**문제점:**
- GPU 큐 간 복잡한 의존성 생성
- 커밋 #1에서 본 것처럼, 큐 간 의존성은 데드락 위험

### 변경 후: CPU 대기 방식

```cpp
void CopyAllocator::submit(CopyCMD cmd)
{
    // 복사 커맨드 실행
    queue->ExecuteCommandLists(1, commandlists);

    // COPY 큐가 완료되면 Fence 신호
    dx12_check(queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled));

#if 1  // CPU 대기 방식 (활성화)
    // CPU에서 즉시 대기 (nullptr = 이벤트 없이 블로킹)
    dx12_check(cmd.fence->SetEventOnCompletion(
        cmd.fenceValueSignaled, nullptr));
#else  // GPU 대기 방식 (비활성화)
    device->queues[QUEUE_GRAPHICS].queue->Wait(...);
    // ...
#endif
}
```

### SetEventOnCompletion의 nullptr 동작

```cpp
// MSDN 문서:
// "If hEvent is nullptr, the function will wait immediately
//  until the fence reaches the specified value."

fence->SetEventOnCompletion(value, nullptr);
// → CPU가 fence 값이 value가 될 때까지 블로킹 대기
```

### 두 방식 비교

#### GPU 대기 방식

```
시간 ───────────────────────────────────────────────▶

CPU:     [submit] [계속 실행...]
                     │
COPY 큐:        [████복사████]
                            │ Signal
                            ▼
GRAPHICS 큐:    ◀─Wait─[█████렌더링█████]

장점: CPU가 블로킹 안됨
단점: GPU 큐 간 의존성 복잡, 데드락 위험
```

#### CPU 대기 방식

```
시간 ───────────────────────────────────────────────▶

CPU:     [submit][대기...][계속 실행...]
                    │          │
COPY 큐:        [████복사████]
                            │
GRAPHICS 큐:              [█████렌더링█████]
                         (의존성 없이 자유롭게 실행)

장점: 단순함, GPU 큐 간 의존성 없음
단점: CPU 블로킹
```

### 왜 CPU 대기가 괜찮은가?

```
CopyAllocator 사용 시점:
├── 게임 시작 시 초기 리소스 로딩
├── 레벨 전환 시 에셋 로딩
├── 스트리밍 텍스처 업로드
└── 런타임 중 동적 리소스 생성

→ 대부분 "로딩 화면" 중에 발생
→ CPU 블로킹해도 사용자가 느끼지 못함
→ 안정성 > 성능
```

---

## 변경 3: 프레임 끝 큐 상호 동기화

### 추가된 코드

```cpp
void GraphicsDevice_DX12::SubmitCommandLists()
{
    // ... 기존 제출 로직 ...

    // 새로 추가: 프레임 끝에서 모든 큐 상호 동기화
    // Sync up every queue to every other queue at the end of the frame:
    // Note: it disables overlapping queues into the next frame
    for (int queue1 = 0; queue1 < QUEUE_COUNT; ++queue1)
    {
        if (queues[queue1].queue == nullptr)
            continue;
        for (int queue2 = 0; queue2 < QUEUE_COUNT; ++queue2)
        {
            if (queue1 == queue2)
                continue;  // 자기 자신은 스킵
            if (queues[queue2].queue == nullptr)
                continue;

            ID3D12Fence* fence = frame_fence[GetBufferIndex()][queue2].Get();
            queues[queue1].queue->Wait(fence, 1);
        }
    }

    // ... 나머지 로직 ...
}
```

### 동작 방식

```
프레임 N 끝:

GRAPHICS ──Wait──▶ COMPUTE의 fence
         ──Wait──▶ COPY의 fence

COMPUTE  ──Wait──▶ GRAPHICS의 fence
         ──Wait──▶ COPY의 fence

COPY     ──Wait──▶ GRAPHICS의 fence
         ──Wait──▶ COMPUTE의 fence
```

**결과:** 프레임 N의 모든 큐가 완료된 후에야 프레임 N+1 시작 가능

### 그림으로 보기

#### 변경 전: 큐 오버랩 가능 (위험)

```
시간 ───────────────────────────────────────────────────────▶

프레임 N:
  GRAPHICS: [███████████████]
  COMPUTE:           [███████████████]
                                    ↑
                              다음 프레임과 겹칠 수 있음
프레임 N+1:
  GRAPHICS:                [███████████████]
                           ↑
                     이전 COMPUTE와 경쟁 조건!
```

#### 변경 후: 큐 오버랩 비활성화 (안전)

```
시간 ───────────────────────────────────────────────────────▶

프레임 N:                    │ 동기화 지점
  GRAPHICS: [███████████████]│
  COMPUTE:           [███████]│
                              │
프레임 N+1:                   │
  GRAPHICS:                   │[███████████████]
  COMPUTE:                    │         [███████]
                              │
                         모든 큐가 완료된 후 다음 프레임
```

### 주석의 의미

```cpp
// Note: it disables overlapping queues into the next frame
```

- **오버랩 비활성화**: 프레임 간 병렬 실행 포기
- **트레이드오프**: 성능 약간 손실, 하지만 안정성 확보
- 커밋 #1에서 제거한 `WaitQueue`와 같은 맥락

---

## 변경 4: 코드 정리

### NULL → nullptr 통일

```cpp
// 변경 전
frame_fence[bufferindex][queue]->SetEventOnCompletion(1, NULL);
fence->SetEventOnCompletion(1, NULL);

// 변경 후
frame_fence[bufferindex][queue]->SetEventOnCompletion(1, nullptr);
fence->SetEventOnCompletion(1, nullptr);
```

**왜 nullptr이 더 좋은가?**

```cpp
// NULL의 정의 (C 스타일)
#define NULL 0  // 또는 ((void*)0)

// nullptr (C++11)
// - 타입 안전: nullptr_t 타입
// - 오버로딩 시 모호성 없음

void func(int);
void func(void*);

func(NULL);     // 어느 함수? 모호함!
func(nullptr);  // void* 버전 호출 (명확)
```

### 지역 변수로 가독성 향상

```cpp
// 변경 전 - 긴 표현식 반복
if (FRAMECOUNT >= BUFFERCOUNT &&
    frame_fence[bufferindex][queue]->GetCompletedValue() < 1)
{
    dx12_check(frame_fence[bufferindex][queue]->SetEventOnCompletion(1, NULL));
}
dx12_check(frame_fence[bufferindex][queue]->Signal(0));

// 변경 후 - 지역 변수 사용
ID3D12Fence* fence = frame_fence[bufferindex][queue].Get();
if (FRAMECOUNT >= BUFFERCOUNT)
{
    if (fence->GetCompletedValue() < 1)
    {
        dx12_check(fence->SetEventOnCompletion(1, nullptr));
    }
}
dx12_check(fence->Signal(0));
```

---

## 커밋 #1과의 연관성

| 커밋 | 핵심 변경 | 공통 주제 |
|------|----------|----------|
| #1 `bf27a9fb` | WaitQueue 제거 | 큐 간 의존성 단순화 |
| #2 `3f5a5cc6` | CopyAllocator CPU 대기, 프레임 끝 동기화 | 큐 간 의존성 단순화 |

**공통 철학:**
1. GPU 큐 간 복잡한 의존성 → 데드락 위험
2. CPU 대기 또는 명시적 동기화 지점 → 안전
3. 약간의 성능 손실보다 안정성 우선

---

## 교훈

### 1. 디버그 이름의 중요성

```
GPU 디버깅 시 이름 없는 객체:
  "Fence at 0x12345678 is blocking"
  → 어떤 펜스인지 모름

이름 있는 객체:
  "frame_fence[QUEUE_GRAPHICS] is blocking"
  → 즉시 문제 파악 가능
```

### 2. GPU 대기 vs CPU 대기

| 방식 | 장점 | 단점 | 적합한 상황 |
|------|------|------|------------|
| GPU 대기 | CPU 프리 | 복잡한 의존성 | 프레임 내 빈번한 동기화 |
| CPU 대기 | 단순함 | CPU 블로킹 | 초기화, 로딩 시점 |

### 3. 프레임 경계의 명확화

```
프레임 간 작업 오버랩:
  장점: 성능 향상
  단점: 경쟁 조건, 디버깅 어려움

프레임 경계 동기화:
  장점: 예측 가능, 안전
  단점: 약간의 성능 손실
```

---

## VizMotive 적용 상태

**적용 완료**: 2026-01-26

### 적용된 항목
- [x] 펜스 명명 (SetName)
- [x] CopyAllocator CPU 대기 방식
- [x] 프레임 끝 큐 상호 동기화
- [x] NULL → nullptr 통일

---

## 관련 커밋

| 커밋 | 내용 |
|------|------|
| `bf27a9fb` (#1) | WaitQueue 제거 - 크로스 큐 의존성 문제의 시작 |
| `93ebdaa6` (#3) | CPU/GPU fence 분리 - 동기화 로직 추가 개선 |
