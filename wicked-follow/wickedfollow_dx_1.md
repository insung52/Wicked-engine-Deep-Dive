# Wicked Engine DX12 변경사항 분석 (2025년 3월~8월)

## 개요
- **분석 기간**: 2025년 3월 1일 ~ 2025년 9월 1일
- **대상 커밋**: DX12 관련 파일 수정 또는 공통 그래픽스 인터페이스 변경
- **목적**: VizMotive Engine 적용 검토
- **분석 대상**: 19개 (비디오 관련 6개 제외)

---

## 커밋 목록 (시간순)

### 비디오 제외 - 분석 대상 (19개)

| # | 날짜 | 커밋 | 제목 | 카테고리 |
|---|------|------|------|----------|
| 1 | 2025-03-12 | `bf27a9fb` | Linux hang fix | 안정성 |
| 2 | 2025-03-15 | `3f5a5cc6` | dx12 and vulkan additional safety updates | 안정성 |
| 3 | 2025-03-16 | `93ebdaa6` | dx12 separated cpu and gpu fences | 안정성 |
| 4 | 2025-03-16 | `dc532889` | xbox gpu hang fix | 안정성 |
| 5 | 2025-03-19 | `652fe0da` | added crossfade fade type | 기능 |
| 6 | 2025-03-21 | `4f503da8` | HDR improvements | 기능 |
| 7 | 2025-03-25 | `99c82676` | dx12 and vulkan improvements | 개선 |
| 8 | 2025-03-26 | `b1b708d2` | camera render texture can be set to material | 기능 |
| 9 | 2025-04-25 | `fe3b922f` | Terrain spline material and some gui updates | 버그수정 |
| 10 | 2025-04-28 | `30917c9e` | GPU buffer suballocator | 성능 |
| 11 | 2025-04-28 | `8582ea3d` | Terrain determinism fixes | 버그수정 |
| 12 | 2025-05-18 | `a948117e` | textured rectangle lights, video frame pacing | 기능 |
| 13 | 2025-05-20 | `e6a003cd` | stringconvert replacement | 리팩토링 |
| 14 | 2025-05-24 | `96f267e1` | improved shadow bias | 렌더링 |
| 15 | 2025-06-03 | `a44d321b` | removal of libraries (basis universal, qoi) | 리팩토링 |
| 16 | 2025-06-14 | `df69a706` | dx12: root signature desc life extender | 안정성 |
| 17 | 2025-06-28 | `6c973af6` | block compressed texture saving fix | 버그수정 |
| 18 | 2025-07-13 | `6cc52406` | prevent clang warning | 빌드 |
| 19 | 2025-08-15 | `e9d5cd09` | texture swizzle improvement | 기능 |

### 비디오 관련 - 분석 제외 (6개)

| 날짜 | 커밋 | 제목 |
|------|------|------|
| 2025-05-03 | `cdb5f8f2` | Video decode fixes for AMD GPU |
| 2025-05-04 | `0dbb81b1` | dx12 video decoder fix |
| 2025-05-04 | `0bfd9485` | DX12 video decoder fix for Nvidia GPU |
| 2025-05-05 | `a5830044` | video: Nvidia buffer workaround, h265 api |
| 2025-05-09 | `aeb71baa` | video small updates |
| 2025-07-11 | `e9f8647f` | re-added custom dxva.h |

---

# 상세 분석 (시간순)

---

## #1. `bf27a9fb` - Linux hang fix ✅ 적용 완료
**날짜**: 2025-03-12 | **카테고리**: 안정성 | **우선순위**: 중간

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/01_bf27a9fb_linux_hang_fix.md)

**요약**: `WaitQueue()` API가 Linux에서 행(hang)을 유발하여 완전 제거됨
- `WaitQueue()` 함수 및 `wait_queues` 벡터 제거
- Wetmap 처리를 별도 컴퓨트 커맨드리스트 대신 동일 커맨드리스트에서 처리하도록 변경

**VizMotive 적용 완료** (2026-01-26)
- `GraphicsDevice_DX12.h/cpp`에서 `WaitQueue` 관련 코드 제거
- `GBackendDevice.h`에서 가상 함수 선언 제거

---

## #2. `3f5a5cc6` - dx12 and vulkan additional safety updates ✅ 적용 완료
**날짜**: 2025-03-15 | **카테고리**: 안정성 | **우선순위**: 높음

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/02_3f5a5cc6_safety_updates.md)

**요약**: DX12/Vulkan 안정성 향상을 위한 3가지 주요 변경
1. **펜스 디버그 이름 설정** - PIX/NSight에서 펜스 식별 가능
2. **CopyAllocator CPU 대기** - GPU 큐 Wait 대신 CPU 블로킹으로 변경
3. **프레임 끝 큐 상호 동기화** - 모든 큐가 서로의 작업 완료를 대기

**VizMotive 적용 완료** (2026-01-26)
- 모든 펜스에 SetName 추가
- CopyAllocator에서 CPU 대기 방식으로 변경
- 프레임 종료 시 큐 간 상호 Wait 로직 추가

---

## #3. `93ebdaa6` - dx12 separated cpu and gpu fences ✅ 적용 완료
**날짜**: 2025-03-16 | **카테고리**: 안정성 | **우선순위**: 높음

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/03_93ebdaa6_separated_fences.md)

**요약**: commit #2와 함께 적용 필수. 단일 펜스를 CPU/GPU 용도별로 분리하여 경쟁 조건 해결
- `frame_fence` → `frame_fence_cpu` + `frame_fence_gpu` 분리
- CPU 펜스: 값 1로 고정, 리셋 가능
- GPU 펜스: FRAMECOUNT로 단조 증가, 리셋 안함

**핵심 원리**: CPU 펜스 리셋이 GPU-GPU Wait에 영향을 주지 않도록 분리

**VizMotive 적용 완료** (2025-01-26)
- `GraphicsDevice_DX12.h`: 펜스 배열 분리
- `GraphicsDevice_DX12.cpp`: 펜스 생성/Signal/Wait 로직 수정

---

## #4. `dc532889` - xbox gpu hang fix ✅ 적용 완료
**날짜**: 2025-03-16 | **카테고리**: 안정성 | **우선순위**: 높음

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/04_dc532889_xbox_gpu_hang_fix.md)

**요약**: Indirect 버퍼 미초기화로 인한 Xbox GPU hang 수정
- **Indirect Buffer**: GPU가 버퍼에서 draw/dispatch 파라미터를 직접 읽는 방식
- **문제**: 미초기화 시 쓰레기값 → 수백만 인스턴스 그리려고 시도 → GPU hang
- **해결**: `CreateBufferZeroed()` 헬퍼 함수로 버퍼 생성 시 0 초기화 강제

**VizMotive 적용 완료** (2025-01-26)
- `ShaderEngine.cpp`, `SortLib.cpp`, `RenderPath3D_Detail.cpp`, `GaussianSplatting_Detail.cpp`
- 모든 `INDIRECT_ARGS` 버퍼를 `CreateBufferZeroed`로 변경

---

## #5. `652fe0da` - added crossfade fade type ⏭️ 미적용 (해당없음)
**날짜**: 2025-03-19 | **카테고리**: 기능 | **우선순위**: 낮음

**요약**: Wicked Application 레벨의 씬 전환 효과 기능. DX12 코어 변경 아님.
- `FadeManager`에 CrossFade 타입 추가 (기존 FadeToColor 외에)
- 씬 전환 시 이전 화면과 새 화면을 직접 블렌딩하는 효과

**VizMotive**: 해당없음
- Wicked의 `Application` / `RenderPath` / `FadeManager` 아키텍처 미사용
- 자체 렌더링 파이프라인 구조로 씬 전환 시스템 불필요

---

## #6. `4f503da8` - HDR improvements ✅ 적용 완료
**날짜**: 2025-03-21 | **카테고리**: 기능 | **우선순위**: 중간

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/06_4f503da8_hdr_improvements.md)

**요약**: HDR10 렌더링 파이프라인 개선 및 SWAPCHAIN ResourceState 추가
- **배경**: HDR10_ST2084는 비선형 색공간이라 선형 블렌딩 시 색상 왜곡 발생
- **DX12 변경**:
  - `ResourceState::SWAPCHAIN = 1 << 17` 추가 (→ `D3D12_RESOURCE_STATE_PRESENT`)
  - `GetBackBuffer()`에서 `desc.layout = SWAPCHAIN` 설정
- **효과**: 스왑체인 배리어 전환 시 정확한 시작/끝 상태 보장

**VizMotive 적용 완료** (2025-01-26)
- `GBackend.h`: `SWAPCHAIN` 상태 추가
- `GraphicsDevice_DX12.cpp`: PRESENT 매핑 및 GetBackBuffer layout 설정
- (HDR10 중간 렌더타겟 로직은 아키텍처 다름으로 미적용)

---

## #7. `99c82676` - dx12 and vulkan improvements ✅ 적용 완료
**날짜**: 2025-03-25 | **카테고리**: 개선 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | `wiGraphicsDevice_DX12.cpp` (-68줄, +54줄), `wiGraphicsDevice_DX12.h` (+2줄, -2줄) |
| 핵심 변경 | CopyAllocator 단순화, StencilRef 캐싱, 프레임 펜스 워크플로우 개선 |

#### 변경 1: CopyAllocator 대폭 단순화

**변경 전**:
```cpp
struct CopyCMD {
    uint64_t fenceValueSignaled = 0;  // 매번 증가
    bool IsCompleted() const { return fence->GetCompletedValue() >= fenceValueSignaled; }
};

CopyCMD CopyAllocator::allocate(uint64_t staging_size) {
    for (i : freelist) {
        if (freelist[i].uploadbuffer.desc.size >= staging_size) {
            if (freelist[i].IsCompleted()) {  // 완료 체크 후 재사용
                cmd = std::move(freelist[i]);
                // ...
            }
        }
    }
}

void CopyAllocator::submit(CopyCMD cmd) {
    locker.lock();
    cmd.fenceValueSignaled++;
    freelist.push_back(cmd);  // 먼저 freelist에 추가
    locker.unlock();

    queue->ExecuteCommandLists(1, commandlists);
    queue->Signal(cmd.fence.Get(), cmd.fenceValueSignaled);

    // GPU 대기 (다른 큐들이 복사 완료 대기)
    device->queues[QUEUE_GRAPHICS].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
    device->queues[QUEUE_COMPUTE].queue->Wait(...);
    // ...
}
```

**변경 후**:
```cpp
struct CopyCMD {
    // fenceValueSignaled 제거
    // IsCompleted() 제거
};

CopyCMD CopyAllocator::allocate(uint64_t staging_size) {
    for (i : freelist) {
        if (freelist[i].uploadbuffer.desc.size >= staging_size) {
            // IsCompleted() 체크 없이 바로 재사용
            cmd = std::move(freelist[i]);
            // ...
        }
    }
    // 새 버퍼 생성 시 최소 크기 보장
    uploaddesc.size = std::max(uploaddesc.size, uint64_t(65536));
}

void CopyAllocator::submit(CopyCMD cmd) {
    cmd.commandList->Close();

    cmd.fence->Signal(0);  // CPU에서 펜스 리셋

    queue->ExecuteCommandLists(1, commandlists);
    queue->Signal(cmd.fence.Get(), 1);  // GPU가 1로 시그널

    cmd.fence->SetEventOnCompletion(1, nullptr);  // CPU 즉시 대기

    std::scoped_lock lock(locker);
    freelist.push_back(cmd);  // 완료 후 freelist에 추가
}
```

**핵심 차이**:
| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 펜스 값 | `fenceValueSignaled++` (증가) | 고정 `0` → `1` |
| 완료 체크 | `IsCompleted()` 호출 | **제거** (항상 완료 상태) |
| 동기화 | GPU Wait (다른 큐 대기) | **CPU Wait** (즉시 블로킹) |
| freelist 추가 | 실행 전 | **실행 완료 후** |
| 최소 버퍼 크기 | 없음 | **65536 바이트** |

#### 변경 2: StencilRef 캐싱

```cpp
// wiGraphicsDevice_DX12.h - CommandList_DX12에 추가
uint32_t prev_stencilref = 0;

// reset()에서 초기화
void reset() {
    // ...
    prev_stencilref = 0;
}

// wiGraphicsDevice_DX12.cpp
void GraphicsDevice_DX12::BindStencilRef(uint32_t value, CommandList cmd)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);
    if (commandlist.prev_stencilref != value)  // 변경된 경우만
    {
        commandlist.prev_stencilref = value;
        commandlist.GetGraphicsCommandList()->OMSetStencilRef(value);
    }
}
```

**효과**: 동일 값 반복 설정 시 DX12 호출 생략 → CPU 오버헤드 감소

#### 변경 3: 프레임 펜스 워크플로우 개선

**초기화 시 펜스 값 설정**:
```cpp
// 펜스 생성 직후
dx12_check(frame_fence_cpu[buffer][queue]->Signal(1)); // 초기값 1 (free 상태)
```

**프레임 제출 시**:
```cpp
// 변경 전
if (FRAMECOUNT >= BUFFERCOUNT && fence->GetCompletedValue() < 1) {
    fence->SetEventOnCompletion(1, nullptr);
}
fence->Signal(0);  // 프레임 끝에서 리셋

// 변경 후
frame_fence_cpu->Signal(0);  // 프레임 시작에서 0으로 (in use)
queue.submit();
queue.queue->Signal(frame_fence_cpu, 1);  // GPU가 1로 (free)
// ...
if (fence->GetCompletedValue() < 1) {  // FRAMECOUNT 체크 제거
    fence->SetEventOnCompletion(1, nullptr);
}
// fence->Signal(0) 제거 - 다음 프레임 시작에서 처리
```

**논리 흐름**:
```
Frame N 시작:
  CPU Signal(fence, 0)     ← "이 버퍼 사용 중"
  GPU 작업 제출
  GPU Signal(fence, 1)     ← "작업 완료, 재사용 가능"

Frame N+BUFFERCOUNT 시작:
  CPU Signal(fence, 0)     ← 새 프레임 시작
  (이전 GPU 작업 미완료 시)
  CPU Wait(fence, 1)       ← GPU 완료 대기
```

#### 변경 4: Device Removed 로깅 개선

```cpp
// 변경 전 (DEBUG만)
#ifdef _DEBUG
    char buff[64] = {};
    sprintf_s(buff, "Device Lost on Present: Reason code 0x%08X\n", ...);
    OutputDebugStringA(buff);
#endif

// 변경 후 (항상)
wilog_messagebox("Device Lost on Present: %s",
    wi::helper::GetPlatformErrorString(...).c_str());
```

---

#### VizMotive 비교

#### 비교 항목

| 항목 | Wicked (99c82676 이후) | VizMotive (현재) | 상태 |
|------|------------------------|------------------|------|
| CopyAllocator fenceValueSignaled | ❌ 제거 | ✅ 사용 중 | **미적용** |
| CopyAllocator IsCompleted() | ❌ 제거 | ✅ 사용 중 | **미적용** |
| CopyAllocator CPU Wait | ✅ CPU 대기 | ❌ GPU Wait | **미적용** |
| StencilRef 캐싱 | ✅ prev_stencilref | ❌ 매번 호출 | **미적용** |
| 펜스 초기값 Signal(1) | ✅ 있음 | ❌ 없음 | **미적용** |
| 프레임 시작 Signal(0) | ✅ 있음 | ❌ 없음 | **미적용** |
| FRAMECOUNT >= BUFFERCOUNT 체크 | ❌ 제거 | ✅ 있음 | **미적용** |

#### VizMotive 코드 현황

**1. CopyAllocator - 구버전 패턴**
```cpp
// VizMotive: GraphicsDevice_DX12.h:95-98
uint64_t fenceValueSignaled = 0;  // 아직 사용 중
inline bool IsCompleted() const { return fence->GetCompletedValue() >= fenceValueSignaled; }

// VizMotive: GraphicsDevice_DX12.cpp:1703
if (freelist[i].IsCompleted())  // 완료 체크 사용 중

// VizMotive: GraphicsDevice_DX12.cpp:1766-1774
// GPU Wait 패턴 사용 중
hr = device->queues[QUEUE_GRAPHICS].queue->Wait(cmd.fence.Get(), cmd.fenceValueSignaled);
hr = device->queues[QUEUE_COMPUTE].queue->Wait(...);
hr = device->queues[QUEUE_COPY].queue->Wait(...);
```

**2. StencilRef - 캐싱 없음**
```cpp
// VizMotive: GraphicsDevice_DX12.cpp:6781-6785
void GraphicsDevice_DX12::BindStencilRef(uint32_t value, CommandList cmd)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);
    commandlist.GetGraphicsCommandList()->OMSetStencilRef(value);  // 매번 호출
}
```

**3. 프레임 펜스 - 구버전 워크플로우**
```cpp
// VizMotive: GraphicsDevice_DX12.cpp:5629-5636
if (FRAMECOUNT >= BUFFERCOUNT && frame_fence[bufferindex][queue]->GetCompletedValue() < 1)
{
    hr = frame_fence[bufferindex][queue]->SetEventOnCompletion(1, NULL);
}
hr = frame_fence[bufferindex][queue]->Signal(0);  // 프레임 끝에서 리셋
```

#### 적용 권장사항

| 우선순위 | 항목 | 이유 |
|----------|------|------|
| **중간** | StencilRef 캐싱 | CPU 오버헤드 감소 (간단한 수정) |
| **중간** | CopyAllocator CPU Wait | 코드 단순화 (GPU Wait 제거) |
| **낮음** | 펜스 워크플로우 변경 | 동작에 영향 없음, 논리 명확성 개선 |

#### 적용 예시: StencilRef 캐싱

```cpp
// GraphicsDevice_DX12.h - CommandList_DX12에 추가
+ uint32_t prev_stencilref = 0;

// reset()에 추가
+ prev_stencilref = 0;

// GraphicsDevice_DX12.cpp - BindStencilRef 수정
void GraphicsDevice_DX12::BindStencilRef(uint32_t value, CommandList cmd)
{
    CommandList_DX12& commandlist = GetCommandList(cmd);
+   if (commandlist.prev_stencilref != value)
+   {
+       commandlist.prev_stencilref = value;
        commandlist.GetGraphicsCommandList()->OMSetStencilRef(value);
+   }
}
```

#### VizMotive 적용 완료 ✅

**적용 일자**: 2025-01-26

**1. CopyAllocator 단순화**

| 파일 | 라인 | 변경 |
|------|------|------|
| `GraphicsDevice_DX12.h` | 95-98 | `fenceValueSignaled`, `IsCompleted()` 제거 |
| `GraphicsDevice_DX12.cpp` | 1700-1712 | `IsCompleted()` 체크 제거 |
| `GraphicsDevice_DX12.cpp` | 1746-1768 | submit: 고정 0→1 패턴, freelist 완료 후 추가 |

```cpp
// submit 변경 내용
dx12_check(cmd.commandList->Close());
dx12_check(cmd.fence->Signal(0)); // Reset to 0
queue->ExecuteCommandLists(1, commandlists);
dx12_check(queue->Signal(cmd.fence.Get(), 1)); // Signal 1
dx12_check(cmd.fence->SetEventOnCompletion(1, nullptr)); // CPU wait
std::scoped_lock lock(locker);
freelist.push_back(cmd); // Add after completion
```

**2. StencilRef 캐싱**

| 파일 | 라인 | 변경 |
|------|------|------|
| `GraphicsDevice_DX12.h` | 177 | `prev_stencilref` 멤버 추가 |
| `GraphicsDevice_DX12.h` | 211 | `reset()`에서 초기화 |
| `GraphicsDevice_DX12.cpp` | 6787-6795 | 값 변경 시에만 호출 |

**효과**:
- CopyAllocator 논리 단순화 (fenceValue 증가 불필요)
- StencilRef 중복 API 호출 방지

---

## #8. `b1b708d2` - camera render texture can be set to material
**날짜**: 2025-03-26 | **카테고리**: 기능 | **우선순위**: 중간

**DX12 변경**:
```cpp
// DISABLE_RENDERPASS 매크로 도입 (Xbox용)
#ifdef PLATFORM_XBOX
#define DISABLE_RENDERPASS
#endif

// RenderPassEnd 개선 - 리졸브 배리어 처리
if (!commandlist.resolve_src_barriers.empty()) {
    commandlist.GetGraphicsCommandList()->ResourceBarrier(...);
}
```

Xbox에서 RenderPass API 비활성화 옵션 추가.

---

## #9. `fe3b922f` - Terrain spline material
**날짜**: 2025-04-25 | **카테고리**: 버그수정 | **우선순위**: 낮음

**커밋 내용**: 터레인 스플라인 머티리얼 기능 + GUI 개선 (31개 파일, +226/-362줄)

**DX12 변경** (4줄만): CopyAllocator 에러 체크 추가
```cpp
// 변경 전
cmd.commandList->Close();
cmd.fence->Signal(0);

// 변경 후
dx12_check(cmd.commandList->Close());
dx12_check(cmd.fence->Signal(0));
```

**VizMotive**: 에러 체크 없이 `hr =` 또는 직접 호출 → 선택적 적용 (디버깅 용이성)

---

## #10. `30917c9e` - GPU buffer suballocator ✅ 적용 완료
**날짜**: 2025-04-28 | **카테고리**: 성능 | **우선순위**: 중간

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | 20개 파일, +993줄 -56줄 |
| 핵심 변경 | 메시 버퍼를 256MB 블록에서 서브할당하여 버퍼 전환 최소화 |

#### WickedEngine 방식의 문제점
- `CreateAliasingResource()`로 별도의 ID3D12Resource 생성
- Debug Layer에서 `D3D12_MESSAGE_ID_HEAP_ADDRESS_RANGE_INTERSECTS_MULTIPLE_BUFFERS` 경고 발생
- 경고 필터링으로 억제 필요

#### VizMotive 적용 완료 ✅

**적용 일자**: 2026-01-27

**VizMotive 개선**: WickedEngine과 달리 **순수 오프셋 기반 뷰 방식** 채택
- `CreateAliasingResource()` 사용하지 않음
- 단일 256MB 블록 버퍼 + 오프셋 기반 SRV/UAV 뷰
- Debug Layer 경고 없이 동작 (경고 필터링 불필요)

#### **상세 구현 문서**: [GPU Buffer Suballocation Without Debug Layer Warnings](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/suballocation_withoutWarning.md)

#### 핵심 변경 요약

| 항목 | WickedEngine | VizMotive |
|------|--------------|-----------|
| 블록 버퍼 플래그 | `ALIASING_BUFFER` | 일반 버퍼 |
| 메시별 Resource | `CreateAliasingResource()` | 없음 (포인터 참조) |
| 데이터 저장 | `generalBufferOffsetAllocationAlias` (복사본) | `blockBufferRef` + `blockBaseOffset` |
| Debug Layer | 경고 필터링 필요 | 경고 없음 |

**변경 파일**:
- `GraphicsDevice_DX12.cpp` - 경고 필터 제거, `UploadToBufferRegion()` 추가
- `GBackend.h` - `BufferSuballocation` 포인터 방식으로 변경
- `GComponents.h` - `GPrimBuffers`에 `blockBufferRef`, `blockBaseOffset`, 헬퍼 메서드 추가
- `Renderer.cpp` - `ALIASING_BUFFER` 플래그 제거, VB/IB 바인딩 로직 수정
- `GeometryComponent.cpp` - 절대 offset 기반 뷰 생성, Raytracing BLAS 수정

**기반 인프라** (2025-01-26 적용):
- `offsetAllocator` 라이브러리 추가 (`EngineCore/ThirdParty/`)
- `PageAllocator` 클래스 추가 (`EngineCore/Utils/Allocator.h`)

---

## #11. `8582ea3d` - Terrain determinism fixes ✅ 적용 완료
**날짜**: 2025-04-28 | **카테고리**: 버그수정 | **우선순위**: 중간

**DX12 변경**: PSO 바인딩 최적화
```cpp
// static PSO 조기 종료 추가
else
    return; // early exit for static pso

// dynamic PSO 캐싱 개선
if (commandlist.prev_pipeline_hash == pipeline_hash) {
    commandlist.active_pso = pso;  // 이 줄 추가!
    return; // early exit for dynamic pso|renderpass
}
```

**VizMotive 비교**:
- VizMotive (`GraphicsDevice_DX12.cpp:6839`)는 early exit 시 `active_pso = pso` 설정 안함 → 잠재적 버그
- static PSO 블록 뒤 `else return;` 없음
- **적용 권장**: 중간 (PSO 상태 일관성)

#### VizMotive 적용 완료 ✅

**적용 일자**: 2026-01-27
**VizMotive 커밋**: `2b18b324`

**적용 내용** (`GraphicsDevice_DX12.cpp`):
```cpp
// 1. static PSO 조기 종료 추가 (line 6843)
else
    return; // early exit for static pso

// 2. dynamic PSO 캐싱 개선 (line 6855-6857)
if (commandlist.prev_pipeline_hash == pipeline_hash) {
    commandlist.active_pso = pso;  // 누락되었던 부분 추가
    return; // early exit for dynamic pso|renderpass
}
```

**수정 효과**:
- PSO 해시가 일치해도 `active_pso`가 갱신되지 않아 발생할 수 있는 불일치 버그 수정
- static PSO 경로에서 불필요한 해시 계산/비교 회피

**참고**: 이 커밋의 본래 목적인 "Terrain determinism"은 `wiNoise.h`의 크로스 플랫폼 결정론적 노이즈 구현 관련이며, DX12 변경은 부수적인 PSO 최적화임

---

## #12. `a948117e` - textured rectangle lights
**날짜**: 2025-05-18 | **카테고리**: 기능 | **우선순위**: 낮음

**DX12 변경**: `wiGraphicsDevice.h`에 헬퍼 추가
```cpp
void CreateMipgenSubresources(Texture& texture);
```

#### VizMotive 해당 없음 ⏭️

**분석 일자**: 2026-01-27

**이유**: VizMotive는 동일한 로직을 이미 인라인으로 구현하고 있음

| 위치 | 텍스처 | 구현 방식 |
|------|--------|----------|
| `Renderer.cpp:197-208` | bloom 2개 | 인라인 루프 |
| `Renderer.cpp:1077-1088` | rtSceneCopy 2개 | 인라인 루프 |
| `Renderer.cpp:1155-1166` | depthBuffer_Copy 2개 | 인라인 루프 |
| `Renderer.cpp:1180-1187` | rtLinearDepth 1개 | 인라인 루프 |

**차이점**:
- WickedEngine: 헬퍼로 텍스처 1개씩 처리
- VizMotive: 2개 텍스처를 한 루프에서 처리 (약간 효율적)

**결론**: 기능적 차이 없음, 코드 정리 수준의 변경이므로 적용 불필요

---

## #13. `e6a003cd` - stringconvert replacement ✅ 적용 완료
**날짜**: 2025-05-20 | **카테고리**: 리팩토링/안정성 | **우선순위**: 중간

**변경 내용**:
1. **버퍼 크기 필수화**: `dest_size_in_characters` 기본값 제거 → 필수 파라미터
2. **커스텀 UTF-8 구현**: Windows API/deprecated std::wstring_convert 제거
3. **크로스 플랫폼**: 모든 플랫폼에서 동일한 동작

```cpp
// 변경 전 - 기본값 -1 (위험)
int StringConvert(const wchar_t* from, char* to, int dest_size = -1);

// 변경 후 - 크기 필수
int StringConvert(const wchar_t* from, char* to, int dest_size);
```

**VizMotive 문제점** (`Helpers.h:87-135`):
1. **버퍼 오버플로우**: `dest_size_in_characters = -1` 기본값 유지
2. **Deprecated API**: `std::wstring_convert` 사용 (C++17에서 deprecated)
3. **UB 버그**: 비-Windows에서 임시 객체 포인터 반환
   ```cpp
   // Line 102 - 버그!
   auto result = cv.from_bytes(from).c_str();  // 임시 객체 → dangling pointer
   ```
4. **플랫폼 차이**: Windows API vs deprecated STL

#### VizMotive 적용 완료 ✅

**적용 일자**: 2026-01-27

**적용 내용**:

1. **`Helpers.h`** - StringConvert 4개 함수 전면 교체 (~250줄)
   - 커스텀 UTF-8 인코딩/디코딩 구현
   - `std::wstring_convert` 제거 (deprecated)
   - Windows API (`MultiByteToWideChar`/`WideCharToMultiByte`) 제거
   - `dest_size_in_characters` 기본값 `-1` 제거 → 필수 파라미터
   - surrogate pair 처리 (U+10000 이상 문자)

2. **`GraphicsDevice_DX12.cpp`** - 호출부 수정
   ```cpp
   // 변경 전 - 버퍼 오버플로우 위험
   vz::helper::StringConvert(name, text)

   // 변경 후 - 크기 명시
   vz::helper::StringConvert(name, text, arraysize(text))
   ```
   - `EventBegin()` (line 7996)
   - `SetMarker()` (line 8014)

**수정 효과**:
- 버퍼 오버플로우 취약점 제거
- Dangling pointer UB 버그 수정
- C++17 deprecated API 제거
- 크로스 플랫폼 결정론적 동작 보장

---

## #14. `96f267e1` - improved shadow bias
**날짜**: 2025-05-24 | **카테고리**: 렌더링 | **우선순위**: 낮음

**DX12 변경**: PSO 스트림에 Flags 필드 추가
```cpp
struct PSO_STREAM1 {
    CD3DX12_PIPELINE_STATE_STREAM_FLAGS Flags;  // 새로 추가
};
// D3D12_PIPELINE_STATE_FLAG_DYNAMIC_DEPTH_BIAS는 Windows 10에서 미작동
```

**VizMotive**: `PSO_STREAM1`에 `Flags` 필드 없음 → Win11 dynamic depth bias 사용시 필요

#### VizMotive 스킵 ⏭️

**분석 일자**: 2026-01-27

**이유**:
- PSO_STREAM1::Flags는 Win10에서 미작동 (주석 처리됨)
- 셰이더 bias 튜닝값은 프로젝트별 조정 필요
- slope_scaled_depth_bias 값 변경은 그림자 품질 튜닝

---

## #15. `a44d321b` - removal of libraries
**날짜**: 2025-06-03 | **카테고리**: 리팩토링 | **우선순위**: 낮음

**DX12 변경**: `dxva.h` 경로 변경 (Utility → 시스템 헤더)

#### VizMotive 해당 없음 ⏭️

**분석 일자**: 2026-01-27

**이유**:
- WickedEngine이 `e9f8647f`에서 다시 로컬 dxva.h 복원
- VizMotive는 이미 로컬 `ThirdParty/dxva.h` 사용 (올바른 접근)
- basis universal/qoi 라이브러리 제거는 VizMotive 구조와 무관

---

## #16. `df69a706` - dx12 root signature desc life extender ✅ 적용 완료
**날짜**: 2025-06-14 | **카테고리**: 안정성 | **우선순위**: 높음

**문제**: PSO가 rootsig_desc 포인터를 쉐이더에서 복사 → 쉐이더 먼저 삭제시 dangling pointer

**해결**:
```cpp
// PipelineState_DX12에 추가
std::shared_ptr<void> rootsig_desc_lifetime_extender;

// PSO 생성시
internal_state->rootsig_desc_lifetime_extender = pso->desc.vs->internal_state;
```

#### VizMotive 적용 완료 ✅

**적용 일자**: 2026-01-27

**적용 내용** (`GraphicsDevice_DX12.cpp`):

1. **PSO internal_state에 필드 추가** (line 1431):
   ```cpp
   std::shared_ptr<void> rootsig_desc_lifetime_extender;
   ```

2. **각 셰이더 타입별 lifetime extender 설정**:
   | 셰이더 | 라인 |
   |--------|------|
   | VS | 4109 |
   | HS | 4121 |
   | DS | 4133 |
   | GS | 4145 |
   | PS | 4157 |
   | MS | 4170 |
   | AS | 4182 |

**수정 효과**:
- 셰이더가 PSO보다 먼저 해제되어도 rootsig_desc 유효
- 셰이더 리로드/핫리로드 시 크래시 방지

---

## #17. `6c973af6` - block compressed texture saving fix ✅ 적용 완료
**날짜**: 2025-06-28 | **카테고리**: 버그수정 | **우선순위**: 중간

**문제**: BC 텍스처 크기가 블록 크기의 배수가 아닐 때 블록 수 계산 오류

```cpp
// 변경 전 - 잘못된 계산 (100/4 = 25)
const uint32_t num_blocks_x = desc.width / pixels_per_block;

// 변경 후 - ceiling division ((100+3)/4 = 26)
const uint32_t num_blocks_x = (mip_width + pixels_per_block - 1) / pixels_per_block;
```

**VizMotive 적용**: `GBackend.h` - `ComputeTextureMemorySizeInBytes()` 수정
- 각 mip 레벨에서 `mip_width`, `mip_height` 먼저 계산
- ceiling division으로 블록 수 계산: `(mip_width + pixels_per_block - 1) / pixels_per_block`
- 기존 코드는 mip 0에서만 블록 수 계산 후 shift → 잘못된 방식

---

## #18. `6cc52406` - prevent clang warning
**날짜**: 2025-07-13 | **카테고리**: 빌드/보안 | **우선순위**: 낮음

**문제**: 포맷 문자열 취약점 (Format String Vulnerability)
```cpp
// 변경 전 - 취약: error에 %s, %n 포함시 문제
wilog_messagebox(error.c_str());

// 변경 후 - 안전: error는 인자로만 사용
wilog_messagebox("%s", error.c_str());
```

**VizMotive**: 해당 없음
- `vz::helper::messageBox(const std::string&)` 사용 → 포맷 문자열 아님
- 직접 문자열 전달 방식으로 취약점 없음

---

## #19. `e9d5cd09` - texture swizzle improvement
**날짜**: 2025-08-15 | **카테고리**: 기능 | **우선순위**: 낮음

**변경**: 스위즐 파싱에서 xyzw 표기법도 인식
```cpp
// rgba 외에 xyzw도 동일하게 처리
case 'x': case 'X': *comp = ComponentSwizzle::R; break;
case 'y': case 'Y': *comp = ComponentSwizzle::G; break;
case 'z': case 'Z': *comp = ComponentSwizzle::B; break;
case 'w': case 'W': *comp = ComponentSwizzle::A; break;
```

**VizMotive**: `SwizzleFromString()` (`GBackend.h:1795`)에서 rgba만 지원
- xyzw 표기법 미지원
- 스크립트/에디터에서 xyzw 사용시 적용 검토
- **적용 권장**: 낮음 (편의 기능)

---

## VizMotive 적용 우선순위

### 높음 (안정성) - 즉시 적용 검토
| # | 커밋 | 내용 | VizMotive 문제 |
|---|------|------|----------------|
| 3 | `93ebdaa6` | CPU/GPU 펜스 분리 | 단일 펜스 → Race condition |
| 4 | `dc532889` | 버퍼 초기화 | Indirect 버퍼 미초기화 |
| ~~16~~ | ~~`df69a706`~~ | ~~Root signature 수명~~ | ✅ **적용 완료** |
| 2 | `3f5a5cc6` | 펜스 명명, 큐 동기화 | 디버깅 어려움 |

### 중간 (성능/기능) - 필요시 적용
| # | 커밋 | 내용 | VizMotive 문제 |
|---|------|------|----------------|
| ~~13~~ | ~~`e6a003cd`~~ | ~~StringConvert 리팩토링~~ | ✅ **적용 완료** |
| ~~17~~ | ~~`6c973af6`~~ | ~~BC 텍스처 블록 계산~~ | ✅ **적용 완료** |
| ~~11~~ | ~~`8582ea3d`~~ | ~~PSO 캐싱 개선~~ | ✅ **적용 완료** |
| 7 | `99c82676` | StencilRef 캐싱 | 불필요한 API 호출 |
| 6 | `4f503da8` | SWAPCHAIN ResourceState | 상태 명시성 부족 |
| ~~10~~ | ~~`30917c9e`~~ | ~~GPU 버퍼 서브할당자~~ | ✅ **적용 완료** |

### 낮음 - 선택적 적용
| # | 커밋 | 내용 |
|---|------|------|
| 1 | `bf27a9fb` | WaitQueue 제거 |
| 5 | `652fe0da` | 헬퍼 함수 (crossfade) |
| ~~12~~ | ~~`a948117e`~~ | ~~CreateMipgenSubresources 헬퍼~~ | ⏭️ **해당 없음** (이미 인라인 구현) |
| ~~14~~ | ~~`96f267e1`~~ | ~~shadow bias 튜닝~~ | ⏭️ **스킵** (Win10 미작동, 튜닝값) |
| ~~15~~ | ~~`a44d321b`~~ | ~~dxva.h 경로 변경~~ | ⏭️ **해당 없음** (WickedEngine 복원됨) |
| 8 | `b1b708d2` | DISABLE_RENDERPASS |
| 9 | `fe3b922f` | dx12_check 에러 체크 |
| 18-19 | 기타 | 빌드/스위즐 |

---

## 다음 단계

(모든 DX12 관련 커밋 적용 완료)

### 적용 완료 커밋 (11개)
| # | 커밋 | 내용 | 적용일 |
|---|------|------|--------|
| 1 | `bf27a9fb` | WaitQueue 제거 | 2025-01-26 |
| 2 | `3f5a5cc6` | DX12/Vulkan 안정성 | 2025-01-26 |
| 3 | `93ebdaa6` | CPU/GPU 펜스 분리 | 2025-01-26 |
| 4 | `dc532889` | Xbox GPU hang fix | 2025-01-26 |
| 6 | `4f503da8` | HDR improvements | 2025-01-26 |
| 7 | `99c82676` | DX12/Vulkan improvements | 2025-01-26 |
| 10 | `30917c9e` | GPU buffer suballocator | 2026-01-28 |
| 11 | `8582ea3d` | PSO 캐싱 개선 (Terrain determinism) | 2026-01-27 |
| 13 | `e6a003cd` | StringConvert 리팩토링 | 2026-01-27 |
| 16 | `df69a706` | Root signature 수명 연장 | 2026-01-27 |
| 17 | `6c973af6` | BC 텍스처 블록 계산 | 2026-01-27 |
