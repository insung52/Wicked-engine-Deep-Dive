# Topic: CopyAllocator (복사 전용 커맨드리스트)

## 개요

CPU → GPU 데이터 업로드를 관리하는 CopyAllocator 시스템의 개선 과정.

## 관련 커밋

| 순서 | 커밋 | 날짜 | 핵심 변경 |
|------|------|------|----------|
| 1 | [dx1 #2 `3f5a5cc6`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/02_3f5a5cc6_safety_updates.md) | 2025-03-15 | GPU Wait → CPU Wait로 변경 |
| 2 | [dx1 #7 `99c82676`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/07_99c82676_dx12_vulkan_improvements.md) | 2025-03-25 | 펜스 패턴 단순화, freelist 완료 후 추가 |
| 3 | dx1 #9 `fe3b922f` | 2025-04-25 | dx12_check 에러 체크 추가 |

---

## CopyAllocator란?

```
┌─────────────────────────────────────────────────────────────────┐
│                      CopyAllocator 역할                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU 메모리 ───────────────▶ GPU 메모리                          │
│  (텍스처, 버퍼 데이터)    Copy Queue    (VRAM에 저장)             │
│                                                                 │
│  사용 시점:                                                      │
│  - 텍스처 로딩 (로딩 화면 중)                                    │
│  - 동적 버퍼 업데이트                                             │
│  - 리소스 초기화 (게임 시작 전)                                   │
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
copyAllocator.submit(cmd);
```

---

## 변화 흐름

### 1단계: GPU Wait → CPU Wait (커밋 #2)

**변경 전: GPU 큐 대기 방식**

```cpp
void CopyAllocator::submit(CopyCMD cmd)
{
    queue->ExecuteCommandLists(1, commandlists);
    queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled);

    // 모든 GPU 큐가 복사 완료를 대기 ❌
    device->queues[QUEUE_GRAPHICS].queue->Wait(cmd.fence.Get(), ...);
    device->queues[QUEUE_COMPUTE].queue->Wait(cmd.fence.Get(), ...);
    device->queues[QUEUE_COPY].queue->Wait(cmd.fence.Get(), ...);
}
```

**문제점**:
- GPU 큐 간 복잡한 의존성 생성
- 큐 간 의존성은 데드락 위험

**변경 후: CPU 대기 방식**

```cpp
void CopyAllocator::submit(CopyCMD cmd)
{
    queue->ExecuteCommandLists(1, commandlists);
    queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled);

    // CPU에서 즉시 대기 ✅
    cmd.fence->SetEventOnCompletion(cmd.fenceValueSignaled, nullptr);
}
```

**타임라인 비교**:

```
[GPU Wait 방식]
Copy Queue:   [복사 작업] ──Signal──▶
                              │
Graphics:     ◀──Wait─────────┘ [렌더링] (의존성)
Compute:      ◀──Wait─────────┘ [계산]   (의존성)

문제: 모든 GPU 큐가 복사에 의존

[CPU Wait 방식]
Copy Queue:   [복사 작업] ──Signal──▶
                              │
CPU:          [대기........] ◀─┘ [다음 작업]

Graphics:     [렌더링] (독립적)
Compute:      [계산]   (독립적)

장점: GPU 큐 간 의존성 없음
```

**왜 CPU 대기가 괜찮은가?**

- 대부분 "로딩 화면" 중에 발생
- 어차피 데이터가 필요해서 복사하는 것
- CPU 블로킹해도 사용자가 느끼지 못함
- 안정성 > 성능

---

### 2단계: 펜스 패턴 단순화 (커밋 #7)

**변경 전: 복잡한 펜스 패턴**

```cpp
struct CopyCMD {
    uint64_t fenceValueSignaled = 0;  // 매번 증가

    bool IsCompleted() const {
        return fence->GetCompletedValue() >= fenceValueSignaled;
    }
};

CopyCMD allocate(uint64_t size) {
    for (auto& cmd : freelist) {
        if (cmd.size >= size && cmd.IsCompleted()) {  // 매번 체크
            return cmd;
        }
    }
}

void submit(CopyCMD cmd) {
    cmd.fenceValueSignaled++;  // 값 증가
    freelist.push_back(cmd);   // 실행 전 추가 (미완료 상태!)

    queue->ExecuteCommandLists(...);
    queue->Signal(cmd.fence, cmd.fenceValueSignaled);
}
```

**문제점**:
1. `fenceValueSignaled`가 계속 증가 → 복잡한 상태 관리
2. `IsCompleted()` 매번 호출 필요
3. freelist에 미완료 cmd 존재 → 동시성 이슈

**변경 후: 단순한 펜스 패턴**

```cpp
struct CopyCMD {
    // fenceValueSignaled 제거!
    // IsCompleted() 제거!
};

CopyCMD allocate(uint64_t size) {
    for (auto& cmd : freelist) {
        if (cmd.size >= size) {  // 완료 체크 불필요!
            return cmd;          // freelist에 있으면 항상 완료
        }
    }
}

void submit(CopyCMD cmd) {
    cmd.fence->Signal(0);  // CPU에서 0으로 리셋

    queue->ExecuteCommandLists(...);
    queue->Signal(cmd.fence, 1);  // GPU가 1로 시그널

    cmd.fence->SetEventOnCompletion(1, nullptr);  // CPU 대기

    freelist.push_back(cmd);  // 완료 후에만 추가 ✅
}
```

**핵심 변경**:

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 펜스 값 | `fenceValueSignaled++` (무한 증가) | 고정 `0` → `1` |
| 완료 체크 | `IsCompleted()` 매번 호출 | **불필요** (freelist = 완료됨) |
| freelist 추가 시점 | 실행 전 (미완료 상태) | **실행 완료 후** |
| 최소 버퍼 크기 | 없음 | **65536 바이트** |

---

### 3단계: 에러 체크 추가 (커밋 #9)

간단한 개선: API 호출에 에러 체크 추가

```cpp
// 변경 전
cmd.commandList->Close();
cmd.fence->Signal(0);

// 변경 후
dx12_check(cmd.commandList->Close());
dx12_check(cmd.fence->Signal(0));
```

---

## 최종 구현

```cpp
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

---

## 핵심 개념 정리

### freelist 패턴

```
CopyCMD 재사용 패턴:

allocate() ──▶ freelist에서 적절한 크기 찾기
                    │
                    ▼
submit() ──────▶ GPU 복사 ──▶ CPU 대기 ──▶ freelist에 반환
                                              │
                                              ▼
allocate() ◀────────────────────────── 재사용!
```

### 고정 펜스 패턴 (0 → 1)

```
┌─────────────────────────────────────────────────────────────────┐
│                    고정 펜스 패턴                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [allocate]   freelist에서 꺼냄 (항상 완료 상태)                 │
│       │                                                         │
│       ▼                                                         │
│  [submit]     ① fence->Signal(0)    ← 리셋                      │
│       │       ② GPU 복사 실행                                   │
│       │       ③ queue->Signal(1)    ← 완료 표시                 │
│       │       ④ CPU Wait(1)         ← 대기                      │
│       │       ⑤ freelist.push       ← 반환                      │
│       ▼                                                         │
│  [다음 allocate]  즉시 재사용 가능                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 교훈

1. **동기화 방식 선택**: 로딩/초기화는 CPU Wait가 더 단순하고 안전
2. **상태 단순화**: 무한 증가보다 고정 값 (0/1) 패턴이 관리하기 쉬움
3. **불변성 보장**: freelist에는 완료된 것만 → allocate에서 체크 불필요
4. **최소 크기 설정**: 작은 할당 반복 방지 (65536 바이트)

---

## VizMotive 적용

모든 커밋 적용 완료:
- CPU Wait 방식
- 고정 0→1 펜스 패턴
- freelist 완료 후 추가
- dx12_check 에러 체크
