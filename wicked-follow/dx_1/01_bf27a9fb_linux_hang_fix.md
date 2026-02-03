# 커밋 #1: bf27a9fb - Linux hang fix

## 기본 정보

| 항목 | 내용 |
|------|------|
| 커밋 해시 | `bf27a9fbe83ef66a552f943c33d2efc578a5bce8` |
| 날짜 | 2025-03-12 |
| 작성자 | Turánszki János |
| 카테고리 | 안정성 (Stability) |
| 우선순위 | 높음 |

## 변경 파일 요약

| 파일 | 변경 |
|------|------|
| wiGraphicsDevice.h | `WaitQueue` 가상 함수 선언 제거 |
| wiGraphicsDevice_DX12.h | `wait_queues` 벡터 + `WaitQueue` 선언 제거 |
| wiGraphicsDevice_DX12.cpp | `WaitQueue` 구현 + SubmitCommandLists 내 처리 로직 제거 |
| wiGraphicsDevice_Vulkan.cpp | 동일한 `WaitQueue` 관련 코드 제거 |
| wiRenderPath3D.cpp | wetmap 처리 로직 변경 (별도 큐 → 같은 큐) |
| wiRenderer.cpp | 관련 동기화 코드 정리 |

---

## 문제 상황

### 증상
Linux에서 WickedEngine 실행 시 프로그램이 멈춤(hang) 현상 발생.
Windows DX12에서는 문제없이 동작하지만, Linux Vulkan 환경에서 데드락 발생.

### 원인
`WaitQueue()` API의 크로스 큐 동기화 로직에서 **순환 의존성(Circular Dependency)** 발생.

---

## 배경 지식: DX12 Command Queue

### Command Queue 종류
DX12/Vulkan에서는 여러 종류의 Command Queue가 GPU에서 **비동기적으로 병렬 실행**됩니다:

```
QUEUE_GRAPHICS   - 렌더링 작업 (Draw calls, Render targets)
QUEUE_COMPUTE    - Compute shader 작업
QUEUE_COPY       - 메모리 복사 작업 (Upload, Readback)
QUEUE_VIDEO_DECODE - 비디오 디코딩 (별도)
```

### 큐 간 동기화 필요성
각 큐가 독립적으로 실행되므로, 한 큐의 결과를 다른 큐에서 사용하려면 **동기화**가 필요합니다.

**예시**: COMPUTE에서 계산한 결과를 GRAPHICS에서 렌더링
```
COMPUTE 큐: [계산 작업] ──signal──▶ 세마포어
                                      │
GRAPHICS 큐: ◀──wait─────────────────┘ [렌더링 작업]
```

---

## WaitQueue의 원래 설계 의도

### 목표: 프레임 간 병렬 처리

WaitQueue는 **프레임 간 작업을 병렬화**하여 GPU 활용도를 높이기 위해 설계되었습니다.

### 주방장-설거지 비유로 이해하기

**등장인물**:
- **주방장 (GRAPHICS 큐)**: 요리 담당 (렌더링)
- **설거지 담당 (COMPUTE 큐)**: 설거지 담당 (wetmap 계산)
- **손님 = 프레임**: 1번 손님 = 프레임 1, 2번 손님 = 프레임 2

**규칙**:
1. 설거지는 해당 손님의 요리가 끝나야 시작 가능 (그릇이 나와야 하니까)
2. 다음 손님 요리는 이전 손님 설거지가 끝나야 시작 가능 (깨끗한 그릇 필요)

### 완전 순차 처리 (느림)

```
시간 ─────────────────────────────────────────────────────────────────▶

주방장: [1번 요리    ]         [2번 요리    ]         [3번 요리    ]
설거지:              [1번 설거지]           [2번 설거지]           [3번 설거지]

        ├──────────┼──────────┼──────────┼──────────┼──────────┼
        0          10         20         30         40         50
```

모든 작업이 직렬로 실행되어 비효율적.

### 의도한 병렬 처리 (빠름)

**핵심 통찰**: 설거지 결과물(깨끗한 그릇)은 **다음 손님 요리 시작할 때** 필요.
→ 1번 설거지는 2번 요리가 그릇을 실제로 필요로 하는 시점까지만 끝나면 됨!

```
시간 ─────────────────────────────────────────────────────────────────▶

주방장: [1번 요리    ][2번 요리    ][3번 요리    ]
                    ↓ 그릇 나옴     ↓ 그릇 나옴
설거지:             [1번 설거지]   [2번 설거지]   [3번 설거지]
                      ↓ 깨끗한 그릇      ↓ 깨끗한 그릇

        ├─────────┼─────────┼─────────┼─────────┼─────────┼
        0         10        20        30        40        50
```

**1번 설거지**와 **2번 요리 초반부**가 동시에 실행 가능!

왜냐하면:
- 2번 요리가 깨끗한 그릇이 **실제로 필요한 시점**은 요리 중반~후반
- 1번 설거지가 그때까지만 끝나면 됨

### 프레임 간 의존성 정리

```
1번 손님 (프레임 1):
  1번 요리 ──────────────▶ 1번 설거지
           (끝나야 시작)

2번 손님 (프레임 2):
  1번 설거지 ─────────────▶ 2번 요리 ──────────────▶ 2번 설거지
            (끝나야 시작)          (끝나야 시작)
```

| 구분 | 의존 관계 | 병렬 가능? |
|------|----------|-----------|
| 같은 프레임 내 | N번 요리 → N번 설거지 | ❌ 불가능 (순차) |
| 프레임 간 | N번 설거지 → N+1번 요리 | ✅ **여기서 병렬!** |

### WaitQueue가 제대로 동작했다면

```
프레임 2 시작:

WaitQueue(cmd, COMPUTE) 호출
→ "COMPUTE 큐에서 실행 중인 작업 끝나면 시작해"

관리자: "COMPUTE 큐 확인!"

COMPUTE 큐 상태:
┌──────────────────────────────────────┐
│ 1번 설거지 (프레임 1) - 실행 중       │  ← 이것만 기다림
└──────────────────────────────────────┘
(2번 설거지는 아직 큐에 없음 - 프레임 2에서 나중에 추가될 예정)

주방장: "1번 설거지 끝나면 2번 요리 시작!" ✅ 정상 동작
```

### 실제 버그: "이전 프레임"과 "현재 프레임" 구분 실패

```
프레임 2 시작:

WaitQueue(cmd, COMPUTE) 호출

관리자: "COMPUTE 큐 확인! 있는 거 다 제출해!"
        (← waitqueue.submit() 호출)

COMPUTE 큐 상태:
┌──────────────────────────────────────┐
│ 1번 설거지 (프레임 1) - 실행 중       │  ← 이것만 기다리려 했는데...
│ 2번 설거지 (프레임 2) - 대기 중       │  ← 이것도 같이 제출됨! 💥
└──────────────────────────────────────┘

2번 설거지: "2번 요리 끝나야 시작하는데요?"
2번 요리: "설거지팀 끝나야 시작하는데요?"

💀 서로 기다림 → 데드락
```

**문제의 핵심**: `waitqueue.submit()`이 큐에 있는 **모든 작업**을 제출해버림.
"이전 프레임 작업"만 기다리려 했는데, "현재 프레임 작업"까지 같이 제출!

### 해결책: 병렬 처리 포기

결국 WickedEngine은 프레임 간 병렬 처리를 포기하고, 같은 큐에서 순차 처리하는 방식으로 변경.

```
변경 후:
주방장: [1번 요리 + 설거지][2번 요리 + 설거지][3번 요리 + 설거지]

→ 약간의 성능 손실이 있지만, 데드락 위험 완전 제거
```

**트레이드오프**: 약간의 성능 손실 vs 데드락 위험 제거 → 안전성 선택

---

## WaitQueue API 분석

### API 목적
```cpp
// "현재 cmd가 제출될 때, wait_for 큐의 작업이 끝날 때까지 기다려라"
virtual void WaitQueue(CommandList cmd, QUEUE_TYPE wait_for) = 0;
```

이것은 **GPU-side 동기화**입니다. CPU가 아니라 GPU가 기다립니다.

### 제거된 구현 코드

#### 1. WaitQueue 함수
```cpp
void GraphicsDevice_DX12::WaitQueue(CommandList cmd, QUEUE_TYPE wait_for)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);
    // (대기할 큐 타입, 새 세마포어) 쌍을 저장
    commandlist.wait_queues.push_back(std::make_pair(wait_for, new_semaphore()));
}
```

호출 시점에는 실제 대기가 발생하지 않고, 나중에 처리할 정보만 저장합니다.

#### 2. CommandList_DX12 구조체
```cpp
struct CommandList_DX12 {
    QUEUE_TYPE queue = {};
    uint32_t id = 0;

    // 제거됨
    wi::vector<std::pair<QUEUE_TYPE, Semaphore>> wait_queues;

    // 유지됨
    wi::vector<Semaphore> waits;    // WaitCommandList용
    wi::vector<Semaphore> signals;  // 다른 cmd가 나를 기다릴 때

    void reset(uint32_t bufferindex) {
        buffer_index = bufferindex;
        wait_queues.clear();  // 제거됨
        waits.clear();
        signals.clear();
        // ...
    }
};
```

#### 3. SubmitCommandLists 내 처리 로직
```cpp
void GraphicsDevice_DX12::SubmitCommandLists()
{
    // ...

    // 의존성 체크 - wait_queues 부분 제거됨
    const bool dependency = !commandlist.signals.empty()
                         || !commandlist.waits.empty()
                         || !commandlist.wait_queues.empty();  // 제거됨

    if (dependency)
    {
        // ⚠️ 문제의 핵심 로직 - 전체 제거됨
        for (auto& wait : commandlist.wait_queues)
        {
            CommandQueue& waitqueue = queues[wait.first];  // 대기할 큐
            const Semaphore& semaphore = wait.second;

            // 대기할 큐를 강제로 제출!
            waitqueue.submit();

            // 그 큐가 완료되면 세마포어 신호
            waitqueue.signal(semaphore);

            // 현재 큐가 그 세마포어를 대기
            queue.wait(semaphore);

            free_semaphore(semaphore);
        }
        commandlist.wait_queues.clear();

        // ... waits, signals 처리 (이건 유지됨)
    }
}
```

---

## 데드락 발생 메커니즘

### WickedEngine의 Wetmap 처리 플로우 (변경 전)

Wetmap은 비 오는 날 젖은 표면 효과를 처리하는 기능입니다.

```cpp
// wiRenderPath3D.cpp - 변경 전 코드

void RenderPath3D::Render()
{
    // 1. GRAPHICS 커맨드 리스트 시작
    CommandList cmd = device->BeginCommandList();

    // 2. ⚠️ 이전 프레임의 COMPUTE 작업 완료 대기 요청
    device->WaitQueue(cmd, QUEUE_COMPUTE);

    // ... 렌더링 작업들 ...

    // 3. Wetmap 처리 (별도 COMPUTE 큐)
    if (scene->IsWetmapProcessingRequired())
    {
        // COMPUTE 큐에서 새 커맨드 리스트 시작
        CommandList wetmap_cmd = device->BeginCommandList(QUEUE_COMPUTE);

        // ⚠️ GRAPHICS cmd 완료 대기 요청
        device->WaitCommandList(wetmap_cmd, cmd);

        // Wetmap 갱신 (COMPUTE)
        wi::renderer::RefreshWetmaps(visibility_main, wetmap_cmd);
    }

    // 4. 제출
    SubmitCommandLists();
}
```

### 시간 순서로 본 실행 흐름

```
시간 ──────────────────────────────────────────────────────────▶

[프레임 N]

1. cmd 생성 (GRAPHICS)
   │
2. WaitQueue(cmd, COMPUTE) 호출
   │  └── cmd.wait_queues에 (COMPUTE, sem_A) 저장
   │
3. 렌더링 작업 기록...
   │
4. wetmap_cmd 생성 (COMPUTE)
   │
5. WaitCommandList(wetmap_cmd, cmd) 호출
   │  └── wetmap_cmd.waits에 sem_B 저장
   │  └── cmd.signals에 sem_B 저장
   │
6. RefreshWetmaps 작업 기록...
   │
7. SubmitCommandLists() 호출
   │
   ├── cmd (GRAPHICS) 제출 처리 시작
   │   │
   │   ├── wait_queues 처리 (COMPUTE 대기)
   │   │   │
   │   │   ├── queues[COMPUTE].submit()
   │   │   │   └── ⚠️ wetmap_cmd 제출됨!
   │   │   │       └── wetmap_cmd는 cmd 완료를 기다리는 중...
   │   │   │
   │   │   ├── queues[COMPUTE].signal(sem_A)
   │   │   │   └── COMPUTE 완료 후 sem_A 신호 (하지만 절대 완료 안됨)
   │   │   │
   │   │   └── queue.wait(sem_A)
   │   │       └── GRAPHICS가 sem_A 대기 (영원히...)
   │   │
   │   └── 💀 데드락!
```

### 그림으로 보는 순환 의존성

```
                    ┌────────────────────────────────────┐
                    │                                    │
                    ▼                                    │
    ┌───────────────────────────────┐                   │
    │      GRAPHICS cmd             │                   │
    │  ┌─────────────────────────┐  │                   │
    │  │ WaitQueue(COMPUTE)      │──┼───────────┐       │
    │  │ "COMPUTE 끝나면 시작"     │  │           │       │
    │  └─────────────────────────┘  │           │       │
    │                               │           │       │
    │  ┌─────────────────────────┐  │           │       │
    │  │ 렌더링 작업들...          │  │           │       │
    │  └─────────────────────────┘  │           │       │
    │                               │           ▼       │
    │  ┌─────────────────────────┐  │    ┌──────────────┴──────────────┐
    │  │ signals: [sem_B]        │──┼───▶│      COMPUTE wetmap_cmd    │
    │  │ "나 끝나면 sem_B 신호"    │  │    │  ┌─────────────────────────┐│
    │  └─────────────────────────┘  │    │  │ waits: [sem_B]          ││
    └───────────────────────────────┘    │  │ "sem_B 오면 시작"        ││
                    ▲                    │  └─────────────────────────┘│
                    │                    │                             │
                    │                    │  ┌─────────────────────────┐│
                    │                    │  │ RefreshWetmaps()        ││
                    │                    │  └─────────────────────────┘│
                    │                    └─────────────────────────────┘
                    │                                    │
                    │                                    │
                    └────────────────────────────────────┘
                           COMPUTE가 GRAPHICS를 기다림
                           GRAPHICS가 COMPUTE를 기다림
                                    💀 데드락!
```

### 왜 `waitqueue.submit()`이 문제인가?

```cpp
for (auto& wait : commandlist.wait_queues)
{
    CommandQueue& waitqueue = queues[wait.first];

    // 이 시점에 waitqueue (COMPUTE)에는 wetmap_cmd가 대기 중
    // wetmap_cmd는 cmd (GRAPHICS)의 완료를 기다리고 있음

    waitqueue.submit();  // ← wetmap_cmd 제출!
    // wetmap_cmd: "나는 GRAPHICS cmd가 끝나야 실행할 수 있어..."

    waitqueue.signal(semaphore);  // ← COMPUTE 끝나면 신호
    // 하지만 COMPUTE는 영원히 안 끝남 (GRAPHICS를 기다리니까)

    queue.wait(semaphore);  // ← GRAPHICS가 COMPUTE 대기
    // GRAPHICS: "COMPUTE 끝날 때까지 기다릴게..."
    // 💀 둘 다 서로를 기다림 → 데드락
}
```

---

## 해결책

### 변경 후 코드

```cpp
// wiRenderPath3D.cpp - 변경 후

void RenderPath3D::Render()
{
    CommandList cmd = device->BeginCommandList();

    // WaitQueue 제거됨! 크로스 큐 의존성 없음

    wi::renderer::ProcessDeferredTextureRequests(cmd);

    // ... 다른 작업들 ...

    // Wetmap을 같은 GRAPHICS cmd에서 처리
    if (scene->IsWetmapProcessingRequired())
    {
        wi::renderer::RefreshWetmaps(visibility_main, cmd);  // 같은 cmd!
    }

    // ... 나머지 렌더링 ...
}
```

### 왜 이게 안전한가?

```
변경 전:
  GRAPHICS ──의존──▶ COMPUTE ──의존──▶ GRAPHICS (순환!)

변경 후:
  GRAPHICS: [작업1] → [Wetmap] → [작업2] → ... (순차 실행, 의존성 없음)
```

같은 Command List 내에서는 작업이 **순차적으로** 실행됩니다.
GPU가 알아서 순서대로 처리하므로 데드락 가능성이 없습니다.

---

## 플랫폼별 차이

### 왜 Linux에서만 문제였나?

| 플랫폼 | 동작 |
|--------|------|
| **Windows DX12** | 드라이버가 데드락 감지하고 타임아웃 처리하거나, 내부 스케줄링이 다르게 동작하여 우연히 문제 회피 |
| **Linux Vulkan** | 세마포어 대기가 더 엄격하게 동작. 타임아웃 없이 무한 대기하여 실제 hang 발생 |

또한 Linux의 스레드 스케줄링 차이로 인해 race condition이 더 자주 발생했을 가능성이 있습니다.

---

## WaitQueue vs WaitCommandList

두 API의 차이를 명확히 이해해야 합니다:

### WaitCommandList (유지됨)
```cpp
void WaitCommandList(CommandList cmd, CommandList wait_for);
```
- **특정 커맨드 리스트**의 완료를 대기
- 명시적인 1:1 의존성
- 제출 순서가 명확함

### WaitQueue (제거됨)
```cpp
void WaitQueue(CommandList cmd, QUEUE_TYPE wait_for);
```
- **큐 전체**의 완료를 대기
- "그 큐에 뭐가 있든 다 끝나면 시작"
- 제출 시점에 강제로 해당 큐를 submit하는 로직이 문제

---

## 교훈

### 1. 크로스 큐 동기화의 위험성
```
규칙: 큐 A가 큐 B를 기다리고, 큐 B의 작업이 큐 A를 기다리면 → 데드락
```

### 2. 가능하면 같은 큐에서 처리
병렬 처리의 이점이 크지 않다면, 같은 큐에서 순차적으로 처리하는 것이 안전합니다.

### 3. 암시적 동기화 주의
`WaitQueue`는 "그 큐의 이전 작업들"이라는 암시적 대상을 가집니다.
명시적인 `WaitCommandList`가 더 예측 가능합니다.

### 4. 플랫폼 테스트 필수
Windows에서 동작해도 Linux/Vulkan에서 문제가 될 수 있습니다.

---

## VizMotive 적용 상태

**적용 완료**: 2026-01-26

### 변경된 파일
- `GBackendDevice.h`: `WaitQueue` 가상 함수 선언 제거
- `GraphicsDevice_DX12.h`: `wait_queues` 벡터 제거
- `GraphicsDevice_DX12.cpp`: `WaitQueue` 구현 및 SubmitCommandLists 내 처리 로직 제거

### 참고
VizMotive의 `Renderer.cpp`에서 `WaitQueue` 호출은 이미 주석 처리되어 있어 추가 수정 불필요.

---

## 관련 커밋

| 커밋 | 내용 |
|------|------|
| `3f5a5cc6` (#2) | CopyAllocator CPU 대기로 변경 - 유사한 동기화 안전성 개선 |
| `93ebdaa6` (#3) | CPU/GPU fence 분리 - 동기화 로직 추가 개선 |
