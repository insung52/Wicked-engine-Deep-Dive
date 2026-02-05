# 커밋 #7: 99c82676 - dx12 and vulkan improvements

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `99c82676` |
| 날짜 | 2025-03-25 |
| 작성자 | Turánszki János |
| 카테고리 | 개선 (Improvement) |
| 우선순위 | 중간 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice_DX12.cpp | -68줄, +54줄 |
| wiGraphicsDevice_DX12.h | +2줄, -2줄 |

## 변경 내용 요약

1. **CopyAllocator 대폭 단순화** - 펜스 패턴 변경, CPU Wait 방식으로 통일
2. **StencilRef 캐싱** - 동일 값 반복 설정 시 API 호출 생략
3. **프레임 펜스 워크플로우 개선** - 초기값 설정, 시작/끝 시점 명확화
4. **Device Removed 로깅 개선** - Release 빌드에서도 에러 표시

---

## 배경 지식: CopyAllocator란?

### 역할

`CopyAllocator`는 **CPU → GPU 데이터 업로드**를 관리하는 시스템입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                      CopyAllocator 역할                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU 메모리 ───────────────▶ GPU 메모리                          │
│  (텍스처, 버퍼 데이터)    Copy Queue    (VRAM에 저장)             │
│                                                                 │
│  사용 시점:                                                      │
│  - 텍스처 로딩                                                   │
│  - 동적 버퍼 업데이트                                             │
│  - 리소스 초기화                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 동작 방식

```cpp
// 1. CopyCMD 할당 (업로드 버퍼 + 커맨드리스트)
CopyCMD cmd = copyAllocator.allocate(dataSize);

// 2. 데이터 복사
memcpy(cmd.uploadBuffer.mappedData, sourceData, dataSize);

// 3. GPU 복사 명령 기록
cmd.commandList->CopyBufferRegion(destBuffer, 0, cmd.uploadBuffer, 0, dataSize);

// 4. 제출 및 대기
copyAllocator.submit(cmd);  // GPU가 복사 완료할 때까지 대기
```

### CopyCMD 구조

```cpp
struct CopyCMD {
    ComPtr<ID3D12CommandAllocator> commandAllocator;
    ComPtr<ID3D12GraphicsCommandList> commandList;
    GPUBuffer uploadBuffer;           // CPU → GPU 업로드용 버퍼
    ComPtr<ID3D12Fence> fence;        // 완료 확인용 펜스
    uint64_t fenceValueSignaled = 0;  // (변경 전) 펜스 시그널 값
};
```

---

## 변경 1: CopyAllocator 대폭 단순화

### 변경 전 - 복잡한 펜스 패턴

```cpp
struct CopyCMD {
    uint64_t fenceValueSignaled = 0;  // 매번 증가하는 펜스 값

    bool IsCompleted() const {
        return fence->GetCompletedValue() >= fenceValueSignaled;
    }
};

CopyCMD CopyAllocator::allocate(uint64_t staging_size)
{
    for (size_t i = 0; i < freelist.size(); ++i)
    {
        // 조건 1: 버퍼 크기가 충분한가?
        if (freelist[i].uploadbuffer.desc.size >= staging_size)
        {
            // 조건 2: 이전 작업이 완료되었는가?
            if (freelist[i].IsCompleted())  // 펜스 체크
            {
                cmd = std::move(freelist[i]);
                freelist.erase(...);
                // 재사용
            }
        }
    }
    // 없으면 새로 생성
}

void CopyAllocator::submit(CopyCMD cmd)
{
    locker.lock();
    cmd.fenceValueSignaled++;  // 펜스 값 증가
    freelist.push_back(cmd);   // 먼저 freelist에 추가 (아직 완료 안됨)
    locker.unlock();

    queue->ExecuteCommandLists(1, commandlists);
    queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled);

    // 다른 GPU 큐들이 복사 완료를 대기
    device->queues[QUEUE_GRAPHICS].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
    device->queues[QUEUE_COMPUTE].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
    device->queues[QUEUE_COPY].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
}
```

**문제점**:
1. `fenceValueSignaled`가 계속 증가 → 복잡한 상태 관리
2. `IsCompleted()` 체크 필요 → allocate()에서 매번 확인
3. GPU Wait → 다른 큐들에 의존성 전파
4. freelist에 미완료 cmd 존재 → 동시성 이슈 가능

### 변경 후 - 단순한 펜스 패턴

```cpp
struct CopyCMD {
    // fenceValueSignaled 제거!
    // IsCompleted() 제거!
};

CopyCMD CopyAllocator::allocate(uint64_t staging_size)
{
    for (size_t i = 0; i < freelist.size(); ++i)
    {
        // 조건: 버퍼 크기가 충분한가? (완료 체크 불필요!)
        if (freelist[i].uploadbuffer.desc.size >= staging_size)
        {
            cmd = std::move(freelist[i]);
            freelist.erase(...);
            // 바로 재사용 (freelist에 있으면 항상 완료 상태)
        }
    }

    // 새 버퍼 생성 시 최소 크기 보장
    uploaddesc.size = std::max(uploaddesc.size, uint64_t(65536));
}

void CopyAllocator::submit(CopyCMD cmd)
{
    dx12_check(cmd.commandList->Close());

    // 고정 펜스 패턴: 0 → 1
    dx12_check(cmd.fence->Signal(0));  // CPU에서 0으로 리셋

    queue->ExecuteCommandLists(1, commandlists);
    dx12_check(queue->Signal(cmd.fence.Get(), 1));  // GPU가 1로 시그널

    // CPU에서 즉시 대기 (완료될 때까지 블로킹)
    dx12_check(cmd.fence->SetEventOnCompletion(1, nullptr));

    // 완료 후에만 freelist에 추가
    std::scoped_lock lock(locker);
    freelist.push_back(cmd);
}
```

### 변경 비교표

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 펜스 값 | `fenceValueSignaled++` (무한 증가) | 고정 `0` → `1` |
| 완료 체크 | `IsCompleted()` 매번 호출 | **불필요** (freelist = 완료됨) |
| 동기화 방식 | GPU Wait (큐 간 의존성) | **CPU Wait** (즉시 블로킹) |
| freelist 추가 시점 | 실행 전 (미완료 상태) | **실행 완료 후** |
| 최소 버퍼 크기 | 없음 | **65536 바이트** |

### 왜 이렇게 바꿨나?

**핵심 통찰**: CopyAllocator는 **즉시 완료를 기다려도 됨**

```
CopyAllocator 사용 시점:
  - 텍스처 로딩 (로딩 화면 중)
  - 초기화 (게임 시작 전)
  - 동적 업데이트 (프레임 중이지만 즉시 필요)

→ 어차피 데이터가 필요해서 복사하는 것
→ CPU가 기다려도 문제없음
→ 복잡한 비동기 관리보다 단순한 동기 방식이 낫다
```

### 타임라인 비교

**변경 전 (GPU Wait)**:
```
시간 ──────────────────────────────────────────────────────────▶

Copy Queue:   [복사 작업] ──Signal(fence, 5)──▶
                                │
Graphics Queue: ◀──Wait(fence, 5)─┘ [렌더링] (복사 완료 대기)
Compute Queue:  ◀──Wait(fence, 5)─┘ [계산]   (복사 완료 대기)

문제: 모든 GPU 큐가 복사에 의존
```

**변경 후 (CPU Wait)**:
```
시간 ──────────────────────────────────────────────────────────▶

Copy Queue:   [복사 작업] ──Signal(fence, 1)──▶
                                │
CPU:          [대기........] ◀──┘ [다음 작업]

Graphics Queue: [렌더링] (독립적으로 진행)
Compute Queue:  [계산]   (독립적으로 진행)

장점: GPU 큐 간 의존성 없음, 논리 단순
```

---

## 변경 2: StencilRef 캐싱

### 배경: Stencil Reference란?

스텐실 테스트에서 비교에 사용하는 **참조 값**입니다:

```cpp
// 스텐실 테스트
if (stencilBuffer[pixel] == stencilRef) {
    // 통과
}
```

렌더링 중 자주 변경되지만, **같은 값을 반복 설정**하는 경우도 많습니다.

### 변경 전 - 매번 API 호출

```cpp
void GraphicsDevice_DX12::BindStencilRef(uint32_t value, CommandList cmd)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);
    commandlist.GetGraphicsCommandList()->OMSetStencilRef(value);  // 항상 호출
}
```

```
DrawCall 1: BindStencilRef(1) → OMSetStencilRef(1)
DrawCall 2: BindStencilRef(1) → OMSetStencilRef(1)  // 같은 값인데 또 호출
DrawCall 3: BindStencilRef(1) → OMSetStencilRef(1)  // 또 호출
DrawCall 4: BindStencilRef(2) → OMSetStencilRef(2)
DrawCall 5: BindStencilRef(2) → OMSetStencilRef(2)  // 같은 값인데 또 호출
```

### 변경 후 - 값 변경 시에만 호출

```cpp
// wiGraphicsDevice_DX12.h - CommandList_DX12에 추가
struct CommandList_DX12 {
    // ...
    uint32_t prev_stencilref = 0;  // 이전 값 캐싱

    void reset() {
        // ...
        prev_stencilref = 0;  // 초기화
    }
};

// wiGraphicsDevice_DX12.cpp
void GraphicsDevice_DX12::BindStencilRef(uint32_t value, CommandList cmd)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);

    if (commandlist.prev_stencilref != value)  // 값이 다를 때만
    {
        commandlist.prev_stencilref = value;
        commandlist.GetGraphicsCommandList()->OMSetStencilRef(value);
    }
}
```

```
DrawCall 1: BindStencilRef(1) → OMSetStencilRef(1)  ✓ 호출
DrawCall 2: BindStencilRef(1) → (스킵, 같은 값)
DrawCall 3: BindStencilRef(1) → (스킵, 같은 값)
DrawCall 4: BindStencilRef(2) → OMSetStencilRef(2)  ✓ 호출 (값 변경)
DrawCall 5: BindStencilRef(2) → (스킵, 같은 값)
```

### 효과

- **CPU 오버헤드 감소**: API 호출 수 감소
- **드라이버 부하 감소**: 불필요한 상태 변경 제거
- **간단한 수정**: 2~3줄 추가로 구현

---

## 변경 3: 프레임 펜스 워크플로우 개선

### 배경: 프레임 펜스의 역할

프레임 펜스는 **GPU가 이전 프레임 작업을 완료했는지** 확인하는 데 사용됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                   더블/트리플 버퍼링                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Buffer 0: [Frame N 렌더링] ──▶ [Present] ──▶ [재사용 대기]      │
│  Buffer 1: [Frame N+1 렌더링] ──▶ [Present] ──▶ [재사용 대기]    │
│  Buffer 2: [Frame N+2 렌더링] ──▶ ...                           │
│                                                                 │
│  Frame N+3에서 Buffer 0 재사용하려면:                            │
│    → Frame N의 GPU 작업이 완료되었는지 확인 필요!                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 변경 전 - 프레임 끝에서 리셋

```cpp
// SubmitCommandLists 마지막 부분
for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    ID3D12Fence* fence = frame_fence_cpu[bufferindex][queue].Get();

    // 조건부 대기
    if (FRAMECOUNT >= BUFFERCOUNT && fence->GetCompletedValue() < 1)
    {
        dx12_check(fence->SetEventOnCompletion(1, nullptr));  // GPU 완료 대기
    }

    dx12_check(fence->Signal(0));  // 프레임 끝에서 0으로 리셋
}
```

**흐름**:
```
Frame N 끝:
  1. (필요시) GPU 완료 대기
  2. fence->Signal(0)  ← 리셋

Frame N+1 시작:
  1. GPU 작업 제출
  2. queue->Signal(fence, 1)  ← GPU가 완료 표시
```

**문제**: 펜스 값의 의미가 시점에 따라 다름
- `0`: "리셋됨" (프레임 끝) 또는 "사용 중" (프레임 진행 중)?
- `1`: "GPU 완료"

### 변경 후 - 프레임 시작에서 리셋

```cpp
// 펜스 생성 시 초기값 설정
dx12_check(frame_fence_cpu[buffer][queue]->Signal(1));  // 초기값 1 (free 상태)

// SubmitCommandLists 시작 부분
for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    ID3D12Fence* fence = frame_fence_cpu[bufferindex][queue].Get();

    // 프레임 시작: 0으로 설정 (사용 중 표시)
    dx12_check(fence->Signal(0));
}

// GPU 작업 제출...
queue.submit();
queue.queue->Signal(frame_fence_cpu[bufferindex][q].Get(), 1);  // GPU 완료 시 1

// SubmitCommandLists 마지막 부분
for (int queue = 0; queue < QUEUE_COUNT; ++queue)
{
    ID3D12Fence* fence = frame_fence_cpu[bufferindex][queue].Get();

    // FRAMECOUNT 체크 제거
    if (fence->GetCompletedValue() < 1)
    {
        dx12_check(fence->SetEventOnCompletion(1, nullptr));  // GPU 완료 대기
    }
    // Signal(0) 제거 - 다음 프레임 시작에서 처리
}
```

**흐름**:
```
Frame N 시작:
  1. fence->Signal(0)  ← "버퍼 사용 중"
  2. GPU 작업 제출
  3. queue->Signal(fence, 1)  ← GPU가 완료 표시

Frame N 끝:
  1. (필요시) GPU 완료 대기

Frame N+1 시작:
  1. fence->Signal(0)  ← "버퍼 사용 중"
  ...
```

### 펜스 값의 의미가 명확해짐

| 펜스 값 | 의미 |
|---------|------|
| `0` | "버퍼 사용 중" (CPU가 프레임 시작 시 설정) |
| `1` | "GPU 작업 완료, 재사용 가능" (GPU가 설정) |

### 타임라인 비교

**변경 전**:
```
Frame N:
         GPU 제출 ──────────────▶ GPU Signal(1) ──▶ CPU Wait(1) ──▶ CPU Signal(0)
         │                                                              │
         └──────────────── "언제 0이 되나?" ────────────────────────────┘
```

**변경 후**:
```
Frame N:
  CPU Signal(0) ──▶ GPU 제출 ──▶ GPU Signal(1) ──▶ CPU Wait(1)
  │                                                      │
  "시작"                                               "끝"

명확한 시작/끝 경계
```

---

## 변경 4: Device Removed 로깅 개선

### 변경 전 - Debug 빌드에서만 출력

```cpp
#ifdef _DEBUG
    char buff[64] = {};
    sprintf_s(buff, "Device Lost on Present: Reason code 0x%08X\n",
              static_cast<unsigned int>(hr));
    OutputDebugStringA(buff);
#endif
```

**문제**: Release 빌드에서 GPU 크래시 시 아무 정보 없음

### 변경 후 - 항상 메시지 박스 표시

```cpp
wilog_messagebox("Device Lost on Present: %s",
    wi::helper::GetPlatformErrorString(static_cast<HRESULT>(hr)).c_str());
```

**장점**:
- Release 빌드에서도 에러 확인 가능
- 사용자에게 문제 상황 알림
- 플랫폼별 에러 문자열로 가독성 향상

---

## VizMotive 적용

### 적용 일자
2025-01-26

### 적용 내용

**1. CopyAllocator 단순화**

| 파일 | 라인 | 변경 |
|------|------|------|
| `GraphicsDevice_DX12.h` | 95-98 | `fenceValueSignaled`, `IsCompleted()` 제거 |
| `GraphicsDevice_DX12.cpp` | 1700-1712 | `IsCompleted()` 체크 제거 |
| `GraphicsDevice_DX12.cpp` | 1746-1768 | submit: 고정 0→1 패턴, freelist 완료 후 추가 |

```cpp
// submit 변경 내용
void CopyAllocator::submit(CopyCMD cmd)
{
    dx12_check(cmd.commandList->Close());
    dx12_check(cmd.fence->Signal(0));  // Reset to 0

    ID3D12CommandList* commandlists[] = { cmd.commandList.Get() };
    queue->ExecuteCommandLists(1, commandlists);
    dx12_check(queue->Signal(cmd.fence.Get(), 1));  // Signal 1

    dx12_check(cmd.fence->SetEventOnCompletion(1, nullptr));  // CPU wait

    std::scoped_lock lock(locker);
    freelist.push_back(cmd);  // Add after completion
}
```

**2. StencilRef 캐싱**

| 파일 | 라인 | 변경 |
|------|------|------|
| `GraphicsDevice_DX12.h` | 177 | `prev_stencilref` 멤버 추가 |
| `GraphicsDevice_DX12.h` | 211 | `reset()`에서 초기화 |
| `GraphicsDevice_DX12.cpp` | 6787-6795 | 값 변경 시에만 호출 |

```cpp
void GraphicsDevice_DX12::BindStencilRef(uint32_t value, CommandList cmd)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);
    if (commandlist.prev_stencilref != value)
    {
        commandlist.prev_stencilref = value;
        commandlist.GetGraphicsCommandList()->OMSetStencilRef(value);
    }
}
```

### 적용하지 않은 부분

| 항목 | 이유 |
|------|------|
| 프레임 펜스 워크플로우 변경 | 기존 로직도 정상 동작, 변경 범위가 큼 |
| Device Removed 로깅 | VizMotive 자체 로깅 시스템 사용 |

---

## 효과

### CopyAllocator 단순화
- 코드 복잡도 감소 (fenceValue 증가 로직 제거)
- freelist 동시성 이슈 제거 (완료 후에만 추가)
- GPU 큐 간 의존성 제거 (CPU Wait로 통일)

### StencilRef 캐싱
- CPU 오버헤드 감소
- 불필요한 드라이버 호출 제거

---

## 요약

| 변경 항목 | 변경 전 | 변경 후 |
|-----------|---------|---------|
| CopyAllocator 펜스 | 무한 증가 | 고정 0→1 |
| CopyAllocator 동기화 | GPU Wait | CPU Wait |
| CopyAllocator freelist | 실행 전 추가 | 완료 후 추가 |
| StencilRef | 매번 호출 | 캐싱 후 필요시만 호출 |
| 프레임 펜스 시작 | 없음 | Signal(0) |
| Device Removed | Debug만 | 항상 표시 |
