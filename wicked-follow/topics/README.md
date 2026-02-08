# Wicked Engine DX12 변경사항 - 주제별 분류

## 개요

Wicked Engine DX12 커밋들을 주제별로 분류하여 전체 흐름을 이해하기 쉽게 정리한 문서들.

- **기간**: 2025년 3월 ~ 2026년 1월
- **총 커밋**: dx_1 19개 + dx_2 25개 = 44개
- **분석 대상**: DX12 관련 변경사항

---

## 주제별 문서

| 주제 | 설명 | 관련 커밋 수 |
|------|------|-------------|
| [Frame Fence & Sync](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_frame_fence_sync.md) | 프레임 펜스 동기화 진화 과정 | 4개 |
| [CopyAllocator](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_copy_allocator.md) | CPU→GPU 복사 시스템 개선 | 3개 |
| [Custom Allocator & shared_ptr](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_allocator_shared_ptr.md) | Block Allocator 기반 커스텀 shared_ptr | 11개 |
| [Texture Operations](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_texture.md) | 텍스처 생성/복사/관리 | 5개 |
| [PSO & Root Signature](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_pso_rootsig.md) | 파이프라인 상태 객체 관련 | 4개 |
| [Struct Optimization](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_struct_optimization.md) | 구조체 메모리 최적화 | 2개 |
| [Bug Fixes & Stability](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/topics/topic_bug_fixes.md) | 개별 버그 수정 | 7개 |

---

## 주제별 커밋 매핑

### 1. Frame Fence & Sync
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx1 #2 `3f5a5cc6` | 2025-03-15 | 큐 상호 동기화, race condition 발생 |
| dx1 #3 `93ebdaa6` | 2025-03-16 | CPU/GPU 펜스 분리 |
| dx1 #7 `99c82676` | 2025-03-25 | 워크플로우 명확화 |
| dx2 #21 `3899e472` | 2026-01-11 | 2배열→1배열+값 추적 단순화 |

### 2. CopyAllocator
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx1 #2 `3f5a5cc6` | 2025-03-15 | GPU Wait → CPU Wait |
| dx1 #7 `99c82676` | 2025-03-25 | 펜스 패턴 0→1 고정, freelist 완료 후 추가 |
| dx1 #9 `fe3b922f` | 2025-04-25 | dx12_check 에러 체크 |

### 3. Custom Allocator & shared_ptr
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx2 #3 `a6adfc12` | 2025-11-02 | BlockAllocator alignment |
| dx2 #4 `359ade8d` | 2025-11-14 | Move semantics |
| dx2 #8 `7a5ea2f6` | 2025-11-22 | **pooled shared_ptr 도입** |
| dx2 #9 `e1dc87e4` | 2025-11-23 | weak_ptr 이중 소멸자 방지 |
| dx2 #10 `16a19429` | 2025-11-23 | destruct_on_zero 기본값 수정 |
| dx2 #11 `90e94ed4` | 2025-11-24 | placement new 허용 |
| dx2 #12 `ffcc3abd` | 2025-11-25 | SharedHeapAllocator |
| dx2 #13 `051bfb60` | 2025-11-26 | self-assign 체크 |
| dx2 #19 `be8c766e` | 2025-12-14 | **weak_ptr race condition 해결** |
| dx2 #22 `43b93085` | 2026-01-15 | CommandList BlockAllocator |
| dx2 #18 `7ecbc25a` | 2025-12-06 | D3D12MemAlloc 업데이트 |

### 4. Texture Operations
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx1 #17 `6c973af6` | 2025-06-28 | BC 텍스처 블록 계산 |
| dx2 #17 `a5162e04` | 2025-12-01 | Shader-readable swapchain |
| dx2 #20 `bf2f369b` | 2025-12-31 | DeleteSubresources RTV/DSV |
| dx2 #24 `2e823bb2` | 2026-01-18 | CPU 텍스처 region copy |
| dx2 #25 `24824a1c` | 2026-01-24 | 텍스처 헬퍼 함수 |

### 5. PSO & Root Signature
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx1 #11 `8582ea3d` | 2025-04-28 | PSO 바인딩 최적화 |
| dx1 #14 `96f267e1` | 2025-05-24 | Dynamic Depth Bias |
| dx1 #16 `df69a706` | 2025-06-14 | Root Signature 수명 연장 |
| dx2 #16 `f0203679` | 2025-11-29 | Root Descriptor offset |

### 6. Struct Optimization
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx2 #7 `93278bea` | 2025-11-20 | vtable 제거, 크기 축소 |
| dx2 #14 `973c2850` | 2025-11-26 | final 키워드 |

### 7. Bug Fixes & Stability
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx1 #1 `bf27a9fb` | 2025-03-12 | WaitQueue 제거 |
| dx1 #4 `dc532889` | 2025-03-16 | Indirect 버퍼 초기화 |
| dx1 #6 `4f503da8` | 2025-03-21 | SWAPCHAIN ResourceState |
| dx1 #13 `e6a003cd` | 2025-05-20 | StringConvert 리팩토링 |
| dx2 #1 `0bdd7a2a` | 2025-10-22 | SRV/UAV 범위 검사 |
| dx2 #6 `ccaaf209` | 2025-11-17 | CopyTexture UB 수정 |
| dx2 #15 `cf6f3434` | 2025-11-26 | Clang 경고 수정 |

### 8. 스킵된 커밋 (8개)
| 커밋 | 이유 |
|------|------|
| dx1 #5 `652fe0da` | crossfade (VizMotive 해당없음) |
| dx1 #8 `b1b708d2` | DISABLE_RENDERPASS (Xbox 전용) |
| dx1 #12 `a948117e` | CreateMipgenSubresources (이미 인라인 구현) |
| dx1 #15 `a44d321b` | dxva.h 경로 (WickedEngine 복원됨) |
| dx1 #18 `6cc52406` | 포맷 문자열 경고 (VizMotive 해당없음) |
| dx1 #19 `e9d5cd09` | texture swizzle xyzw (선택적) |
| dx2 #2 `f807d9f6` | GPU vendor preference (편의 기능) |
| dx2 #5 `acf8006e` | compile optimizations (빌드 설정) |
| dx2 #23 `5a067ed2` | console updates (Xbox 전용) |

### 9. 기타 (GPU Buffer Suballocator)
| 커밋 | 날짜 | 내용 |
|------|------|------|
| dx1 #10 `30917c9e` | 2025-04-28 | 256MB 블록 서브할당 |

별도 문서: [suballocation_withoutWarning.md](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/suballocation_withoutWarning.md)

---

## 읽는 순서 추천

1. **Frame Fence & Sync** - DX12 동기화 기본 이해
2. **CopyAllocator** - 간단한 시스템의 개선 과정
3. **Texture Operations** - DX12 텍스처 처리 심화
4. **Custom Allocator & shared_ptr** - 커스텀 메모리 관리 (가장 복잡)
5. **나머지** - 필요에 따라

---

## 상세 문서 위치

- 개별 커밋 분석: `../dx_1/`, `../dx_2/`
- 시간순 요약: `../wickedfollow_dx_1.md`, `../wickedfollow_dx_2.md`
