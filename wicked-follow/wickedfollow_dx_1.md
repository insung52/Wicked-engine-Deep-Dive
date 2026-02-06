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

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/07_99c82676_dx12_vulkan_improvements.md)

**요약**: DX12/Vulkan 코드 개선 및 단순화
1. **CopyAllocator 단순화**
   - 펜스 패턴: `fenceValueSignaled++` → 고정 `0→1`
   - 동기화: GPU Wait → CPU Wait
   - freelist: 실행 전 추가 → 완료 후 추가
2. **StencilRef 캐싱** - 동일 값 반복 설정 시 API 호출 생략
3. **프레임 펜스 워크플로우** - 시작/끝 시점 명확화
4. **Device Removed 로깅** - Release에서도 에러 표시

**VizMotive 적용 완료** (2025-01-26)
- CopyAllocator: 고정 0→1 패턴, CPU Wait, freelist 완료 후 추가
- StencilRef: `prev_stencilref` 캐싱 추가

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

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/11_8582ea3d_pso_binding_optimization.md)

**요약**: PSO(Pipeline State Object) 바인딩 최적화 및 버그 수정
- **Static PSO 조기 종료**: `else return` 추가로 불필요한 해시 계산 회피
- **Dynamic PSO 캐시 히트**: `active_pso = pso` 갱신 추가로 상태 일관성 보장
- **문제**: 해시 캐시 히트 시 `active_pso` 미갱신 → 다른 PSO 전환 후 잘못된 참조 가능

**VizMotive 적용 완료** (2026-01-27, 커밋 `2b18b324`)
- `GraphicsDevice_DX12.cpp`: static PSO 조기 종료 + dynamic PSO active_pso 갱신

**참고**: 커밋 본래 목적은 Terrain determinism (wiNoise.h)이며, DX12 변경은 부수적임

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

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/13_e6a003cd_stringconvert_replacement.md)

**요약**: StringConvert 함수 전면 교체 (보안/안정성 개선)
- **버퍼 오버플로우**: `dest_size = -1` 기본값 → 필수 파라미터화
- **Deprecated API**: `std::wstring_convert` → 커스텀 UTF-8 인코딩/디코딩 구현
- **Dangling Pointer**: `cv.from_bytes(from).c_str()` → 임시 객체 제거, 직접 버퍼 쓰기

**VizMotive 적용 완료** (2026-01-27)
- `Helpers.h`: StringConvert 4개 함수 전면 교체 (~250줄)
- `GraphicsDevice_DX12.cpp`: 호출부에 크기 파라미터 추가

---

## #14. `96f267e1` - improved shadow bias ⏭️ 미적용 (향후 검토)
**날짜**: 2025-05-24 | **카테고리**: 렌더링 | **우선순위**: 낮음

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/14_96f267e1_dynamic_depth_bias.md)

**요약**: Dynamic Depth Bias 지원을 위한 PSO Flags 필드 추가
- **Dynamic Depth Bias**: 런타임에 depth bias 변경 가능 (PSO 재생성 없이)
- **장점**: PSO 폭발 방지, FLOAT 타입 정밀도, 유연한 섀도우 품질 조정
- **요구사항**: Agility SDK 1.608.0+, ID3D12GraphicsCommandList9
- **Wicked Engine**: Flags 필드만 추가, 실제 사용 안 함

**VizMotive 미적용** (향후 검토)
- Static depth bias로 충분히 동작
- 섀도우 품질 개선 필요 시 적용 검토

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

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/16_df69a706_rootsig_lifetime_extender.md)

**요약**: Dangling Pointer 방지를 위한 메모리 수명 연장
- **문제**: PSO가 셰이더의 `rootsig_desc` 포인터를 복사 → 셰이더 먼저 삭제 시 dangling pointer
- **원인**: `rootsig_desc`는 셰이더의 `rootsig_deserializer`가 소유한 메모리를 가리킴
- **해결**: `shared_ptr<void>`로 셰이더 internal_state 참조 → 메모리 수명 연장
- **효과**: 셰이더 핫리로드/정리 시 크래시 방지

**VizMotive 적용 완료** (2026-01-27)
- `PipelineState_DX12`에 `rootsig_desc_lifetime_extender` 필드 추가
- VS, HS, DS, GS, PS, MS, AS 각 셰이더별 lifetime extender 설정

---

## #17. `6c973af6` - block compressed texture saving fix ✅ 적용 완료
**날짜**: 2025-06-28 | **카테고리**: 버그수정 | **우선순위**: 중간

### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/17_6c973af6_bc_texture_block_calculation.md)

**요약**: BC 텍스처 mip 레벨별 블록 수 계산 버그 수정
- **문제**: mip 0 블록 수를 shift하는 방식 → 나머지 픽셀 누락
- **해결**: 각 mip에서 픽셀 크기 계산 → ceiling division으로 블록 수 계산
- **핵심 공식**: `num_blocks = (mip_pixels + block_size - 1) / block_size`
- **증상**: 텍스처 저장 시 하위 mip 데이터 누락, LOD 전환 시 아티팩트

**VizMotive 적용 완료** (2026-01-27)
- `GBackend.h`: `ComputeTextureMemorySizeInBytes()` 함수 수정

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
