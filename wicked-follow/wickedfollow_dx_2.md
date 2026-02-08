# Wicked Engine DX12 변경사항 분석 (2025년 9월~2026년 1월)

## 개요
- **분석 기간**: 2025년 9월 1일 ~ 2026년 1월 28일
- **대상 커밋**: DX12 관련 파일 수정 또는 공통 그래픽스 인터페이스 변경
- **목적**: VizMotive Engine 적용 검토
- **분석 대상**: 25개 (DX12 + Graphics 인터페이스)

---

## 커밋 목록 (시간순)

### 분석 대상 - DX12/Graphics 인터페이스 (25개)

| # | 날짜 | 커밋 | 제목 | 카테고리 |
|---|------|------|------|----------|
| 1 | 2025-10-22 | `0bdd7a2a` | Fix out of bounds crash in SRV and UAV subresource vectors | 버그수정 |
| 2 | 2025-10-26 | `f807d9f6` | added ability to choose gpu by vendor preference | 기능 |
| 3 | 2025-11-02 | `a6adfc12` | block allocator should respect alignment of items | 버그수정 |
| 4 | 2025-11-14 | `359ade8d` | optimizations for hair particle system | 최적화 |
| 5 | 2025-11-17 | `acf8006e` | compile optimizations | 최적화 |
| 6 | 2025-11-17 | `ccaaf209` | get rid of some undefined behaviour | 안정성 |
| 7 | 2025-11-20 | `93278bea` | removed virtual tables from gfx objects, struct size reductions | 최적화 |
| 8 | 2025-11-22 | `7a5ea2f6` | pooled shared ptr | 최적화 |
| 9 | 2025-11-23 | `e1dc87e4` | prevent calling destructor when original refcount was 0 | 버그수정 |
| 10 | 2025-11-23 | `16a19429` | fix typo in #1322 | 버그수정 |
| 11 | 2025-11-24 | `90e94ed4` | placement new is allowed for gfx objects | 리팩토링 |
| 12 | 2025-11-25 | `ffcc3abd` | custom shared_ptr updates | 리팩토링 |
| 13 | 2025-11-26 | `051bfb60` | handle self-assign | 버그수정 |
| 14 | 2025-11-26 | `973c2850` | made some internal structures final | 최적화 |
| 15 | 2025-11-26 | `cf6f3434` | some clang warning fixes | 빌드 |
| 16 | 2025-11-29 | `f0203679` | dx12: root descriptor can use buffer offset | 기능 |
| 17 | 2025-12-01 | `a5162e04` | added support for shader-readable swapchain | 기능 |
| 18 | 2025-12-06 | `7ecbc25a` | gfx allocators update | 업데이트 |
| 19 | 2025-12-14 | `be8c766e` | fixed critical issue with weak_ptr | 버그수정 |
| 20 | 2025-12-31 | `bf2f369b` | dx12 fix: DeleteSubresources didn't handle rtvs and dsvs | 버그수정 |
| 21 | 2026-01-11 | `3899e472` | Mac OS support | 플랫폼 |
| 22 | 2026-01-15 | `43b93085` | dx12, vulkan: using block allocator for command lists | 최적화 |
| 23 | 2026-01-17 | `5a067ed2` | console updates and fixes | 버그수정 |
| 24 | 2026-01-18 | `2e823bb2` | region copy support for CPU textures | 기능 |
| 25 | 2026-01-24 | `24824a1c` | texture upload improvements | 최적화 |

---

## 상세 분석

### #1. `0bdd7a2a` - Fix out of bounds crash in SRV and UAV subresource vectors ✅ 적용 완료
**날짜**: 2025-10-22 | **카테고리**: 버그수정 | **우선순위**: 높음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/01_0bdd7a2a_srv_uav_bounds_check.md)

**요약**: `GetDescriptorIndex()`에서 subresource 벡터 범위 검사 추가
- **문제**: `subresources_srv[subresource]` 접근 시 범위 검사 없음 → 잘못된 인덱스로 크래시
- **해결**: `size()` 비교 후 범위 벗어나면 -1 반환
- **배경 지식**: Subresource (mip level × array slice), SRV/UAV View 개념

**VizMotive 적용 완료** (2026-01-28)
- `GraphicsDevice_DX12.cpp`: SRV/UAV subresource 벡터 범위 검사 추가

---

### #2. `f807d9f6` - added ability to choose gpu by vendor preference ⏭️ 스킵
**날짜**: 2025-10-26 | **카테고리**: 기능 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp |
| 핵심 변경 | `GPUPreference` enum에 `Nvidia`, `Intel`, `AMD` 추가 |

#### 변경 내용
```cpp
// enum 확장
enum class GPUPreference
{
    Discrete,
    Integrated,
    Nvidia,    // 추가
    Intel,     // 추가
    AMD,       // 추가
};
```

#### 스킵 사유
- 멀티 GPU 시스템에서 특정 벤더 GPU 선택 기능
- VizMotive에서 현재 필요하지 않은 편의 기능
- 필요시 나중에 적용 가능

---

### #3. `a6adfc12` - block allocator should respect alignment of items ✅ 이미 적용됨
**날짜**: 2025-11-02 | **카테고리**: 버그수정 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/03_a6adfc12_block_allocator_alignment.md)

**요약**: BlockAllocator에서 T 타입의 메모리 정렬(alignment) 보장
- **문제**: `uint8_t[]` 배열은 1바이트 정렬만 보장 → T 타입 정렬 위반 가능
- **증상**: 성능 저하, atomic 연산 실패, 일부 플랫폼 크래시
- **해결**: `alignas(alignof(T)) RawStruct` 래퍼로 정렬 보장
- **배경 지식**: alignof, alignas, 메모리 정렬 개념

**VizMotive 상태**: ✅ 이미 적용됨 (`Allocator.h:26-32`)

---

### #4. `359ade8d` - optimizations for hair particle system ✅ 이미 적용됨
**날짜**: 2025-11-14 | **카테고리**: 최적화 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/04_359ade8d_move_semantics_const.md)

**요약**: Allocator에서 Move Semantics 최적화 및 const 정확성 추가
- **변경 1**: Move 생성자/대입에서 `std::move` 사용 → 불필요한 복사/atomic 연산 방지
- **변경 2**: `is_empty()`에 `const` 추가 → const 객체 호출 가능, 최적화 기회
- **배경 지식**: lvalue vs rvalue, Move semantics, const 정확성

**VizMotive 상태**: ✅ 이미 적용됨 (`Allocator.h`)

---

### #5. `acf8006e` - compile optimizations ⏭️ 스킵
**날짜**: 2025-11-17 | **카테고리**: 최적화 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | 빌드 설정, wiGraphicsDevice.h/DX12.h/Vulkan.h |
| 핵심 변경 | 예외 제거, RTTI 비활성화, `GetTag()` 함수 추가 |

#### Graphics 관련 변경
```cpp
// 디버깅용 식별자 함수 추가
virtual const char* GetTag() const { return ""; }
const char* GetTag() const override { return "[DX12]"; }
```

#### 스킵 사유
- 빌드 설정 변경은 VizMotive 프로젝트 구조가 다름
- `GetTag()`는 디버깅용 편의 기능
- 필요시 나중에 적용 가능

---

### #6. `ccaaf209` - get rid of some undefined behaviour ✅ 적용 완료
**날짜**: 2025-11-17 | **카테고리**: 안정성 | **우선순위**: 높음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/06_ccaaf209_undefined_behavior_fix.md)

**요약**: `CopyTexture()`에서 잘못된 타입 캐스팅으로 인한 Undefined Behavior 수정
- **문제**: `Texture*`를 `GPUBuffer*`로 C-style 캐스팅 → Strict Aliasing Rule 위반
- **위험**: 우연히 동작하거나, 최적화 시 이상 동작, 다른 플랫폼에서 크래시
- **해결**: 올바른 타입의 `to_internal()` 오버로드 호출
- **배경 지식**: C-style 캐스팅, Strict Aliasing Rule, Undefined Behavior

**VizMotive 적용 완료** (2026-01-28)
- `GraphicsDevice_DX12.cpp`: `to_internal(src)` 직접 호출 (캐스팅 제거)

---

### #7. `93278bea` - removed virtual tables from gfx objects, struct size reductions ✅ 적용 완료
**날짜**: 2025-11-20 | **카테고리**: 최적화 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/07_93278bea_struct_size_optimization.md)

**요약**: 구조체 크기 최적화 및 virtual table 제거로 메모리 절약
- **변경 1**: `BindFlag : uint8_t` - enum 크기 4바이트 → 1바이트
- **변경 2**: `GPUBufferDesc` 멤버 순서 재배치 + `alignment` uint32_t로 축소
- **변경 3**: `GraphicsDeviceChild` 기본 클래스 제거 → vtable 8바이트/객체 절약
- **변경 4**: `final` 키워드 + `operator delete` 금지 (스택/정적 할당만 허용)
- **변경 5**: `GetMinOffsetAlignment` 반환 타입 uint64_t → uint32_t
- **배경 지식**: 구조체 패딩, vtable 오버헤드, devirtualization

**VizMotive 적용 완료** (2026-01-28)
- `GBackend.h`: BindFlag/GPUBufferDesc/TextureDesc 최적화, GraphicsDeviceChild 제거
- `GBackendDevice.h`, `GraphicsDevice_DX12.h`: GetMinOffsetAlignment 반환 타입 변경

---

### #8. `7a5ea2f6` - pooled shared ptr ✅ 적용 완료
**날짜**: 2025-11-22 | **카테고리**: 최적화 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/08_7a5ea2f6_pooled_shared_ptr.md)

**요약**: `std::shared_ptr` 대신 Block Allocator 기반 pooled shared_ptr 사용
- **문제**: Graphics 객체 빈번한 생성/삭제 → 힙 할당 오버헤드, 메모리 단편화
- **해결**: 같은 타입 객체를 연속 메모리에 풀링, free_list로 O(1) 할당/해제
- **DLL 문제**: WickedEngine handle 방식은 `inline` 전역 변수가 DLL별로 별도 인스턴스화되어 실패
- **VizMotive 해결**: allocator 포인터 직접 저장 (8바이트 → 16바이트, DLL 경계 정상 작동)
- **성능**: 할당/해제 10-20배 빠름, 캐시 효율 향상

**VizMotive 적용 완료** (2026-01-28)
- `Allocator.h`: SharedBlockAllocator, shared_ptr, weak_ptr, make_shared 구현
- `GBackend.h`: internal_state 타입 변경
- `GraphicsDevice_DX12.cpp`: make_shared 호출 변경

---

### #9. `e1dc87e4` - prevent calling destructor when original refcount was 0 ✅ 적용 완료
**날짜**: 2025-11-23 | **카테고리**: 버그수정 | **우선순위**: 높음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/09_e1dc87e4_weak_ptr_double_destruct.md)

**요약**: `weak_ptr::lock()`에서 이중 소멸자 호출 방지
- **문제**: lock() 실패 시 inc→dec 과정에서 dec가 소멸자 재호출 → UB
- **해결**: `dec_refcount(ptr, destruct_on_zero)` 파라미터 추가, 롤백 시 `false` 전달
- **한계**: 멀티스레드 race condition 여전히 존재 (커밋 #19에서 완전 해결)

**VizMotive 적용 완료** (2026-01-28)
- `Allocator.h`: dec_refcount 시그니처 및 구현 수정

---

### #10. `16a19429` - fix typo in #1322 ✅ 적용 완료
**날짜**: 2025-11-23 | **카테고리**: 버그수정 | **우선순위**: 높음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/10_16a19429_destruct_on_zero_typo.md)

**요약**: 커밋 #9의 `destruct_on_zero` 기본값 오류 수정
- **문제**: 기본값이 `false`로 설정됨 → 모든 객체 소멸자 호출 안 됨 → 메모리 누수
- **해결**: 기본값을 `true`로 수정 (대부분의 경우 소멸자 호출 필요)
- **참고**: 커밋 #19에서 이 파라미터 자체가 제거됨

**VizMotive 적용 완료** (2026-01-28) - 커밋 #9와 함께 올바른 기본값으로 적용

---

### #11. `90e94ed4` - placement new is allowed for gfx objects ✅ 적용 완료
**날짜**: 2025-11-24 | **카테고리**: 리팩토링 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/11_90e94ed4_placement_new_allowed.md)

**요약**: 커밋 #7에서 과도하게 금지된 placement new 허용
- **문제**: 커밋 #7이 모든 new를 금지 → Block Allocator에서 객체 생성 불가
- **해결**: placement new만 허용 (메모리 할당 없이 생성자만 호출)
- **대상**: GPUBuffer, Texture, RaytracingAccelerationStructure
- **배경 지식**: placement new vs 일반 new, Block Allocator 패턴

**VizMotive 적용 완료** (2026-01-28)
- `GBackend.h`: 세 클래스에 placement new 허용

---

### #12. `ffcc3abd` - custom shared_ptr updates ✅ 부분 적용
**날짜**: 2025-11-25 | **카테고리**: 리팩토링 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/12_ffcc3abd_shared_heap_allocator.md)

**요약**: 단일 객체용 SharedHeapAllocator 추가
- **이름 변경**: SharedBlockAllocator → SharedAllocator (VizMotive 스킵)
- **새 기능**: `SharedHeapAllocator<T>` - 풀링 없이 개별 힙 할당
- **새 함수**: `make_shared_single<T>()` - 한 번만 생성되는 객체용
- **적용 대상**: AllocationHandler (디바이스당 1개 → 블록 풀링 비효율)
- **배경 지식**: Block Allocator vs Heap Allocator 선택 기준

**VizMotive 부분 적용** (2026-01-28)
- `Allocator.h`: SharedHeapAllocator + make_shared_single 추가
- `GraphicsDevice_DX12.cpp`: AllocationHandler에 적용
- 이름 변경은 스킵 (DLL 경계 대응으로 다른 구조 사용)

---

### #13. `051bfb60` - handle self-assign ✅ 이미 적용됨
**날짜**: 2025-11-26 | **카테고리**: 버그수정 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/13_051bfb60_self_assign_check.md)

**요약**: `PageAllocator::Allocation::operator=`에 self-assign 체크 추가
- **문제**: `a = a;` 시 Reset() 후 해제된 리소스 접근 → UB
- **해결**: `if (&other == this) return *this;`

**VizMotive 상태**: ✅ 이미 적용됨 (`Allocator.h:161`)

---

### #14. `973c2850` - made some internal structures final ✅ 적용 완료
**날짜**: 2025-11-26 | **카테고리**: 최적화 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/14_973c2850_internal_struct_final.md)

**요약**: DX12 내부 구조체에 `final` 키워드 추가
- **대상**: `Texture_DX12`, `BVH_DX12`
- **효과**: 컴파일러 devirtualization → 가상 함수 직접 호출 + 인라이닝 가능
- **배경 지식**: final 키워드, devirtualization 최적화

**VizMotive 적용 완료** (2026-01-28)
- `GraphicsDevice_DX12.cpp`: 두 구조체에 final 추가

---

### #15. `cf6f3434` - some clang warning fixes ✅ 적용 완료
**날짜**: 2025-11-26 | **카테고리**: 빌드 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/15_cf6f3434_clang_warning_fix.md)

**요약**: Clang 컴파일러 경고 수정
- **문제**: switch문에 default case 누락 → Clang `-Wswitch-default` 경고
- **해결**: `default: break;` 추가
- **목적**: 크로스 플랫폼 빌드 시 모든 컴파일러에서 경고 0 유지

**VizMotive 적용 완료** (2026-01-28)
- `GraphicsDevice_DX12.cpp`: `_ConvertFormat` switch문에 default 추가

---

### #16. `f0203679` - dx12: root descriptor can use buffer offset ✅ 적용 완료
**날짜**: 2025-11-29 | **카테고리**: 기능 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/16_f0203679_root_descriptor_offset.md)

**요약**: Root Descriptor 바인딩 시 buffer offset 지원
- **문제**: Subresource 바인딩 시 offset 무시 → 항상 버퍼 시작 주소 사용
- **해결**: `SingleDescriptor`에 `buffer_offset` 저장, 바인딩 시 GPU 주소에 더함
- **적용 대상**: SRV/UAV root descriptor
- **배경 지식**: Root Descriptor vs Descriptor Table, Buffer Suballocation

**VizMotive 적용 완료** (2026-01-28)
- `GraphicsDevice_DX12.cpp`: buffer_offset 멤버 추가 및 바인딩 로직 수정

---

### #17. `a5162e04` - added support for shader-readable swapchain ✅ 적용 완료
**날짜**: 2025-12-01 | **카테고리**: 기능 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/17_a5162e04_shader_readable_swapchain.md)

**요약**: SwapChain back buffer를 쉐이더에서 읽을 수 있도록 SRV 지원 추가
- **변경 1**: `DXGI_USAGE_SHADER_INPUT` 플래그 추가
- **변경 2**: Back buffer를 완전한 Texture_DX12 객체로 관리 (RTV + SRV)
- **변경 3**: GetBackBuffer()에서 미리 생성된 객체 재사용
- **효과**: 후처리, 화면 캡처 등에서 back buffer 직접 읽기 가능

**VizMotive 적용 완료** (2026-01-29)
- `GraphicsDevice_DX12.cpp`: SwapChain 구조 변경, SRV 생성 추가

---

### #18. `7ecbc25a` - gfx allocators update ✅ 적용 완료
**날짜**: 2025-12-06 | **카테고리**: 업데이트 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/18_7ecbc25a_d3d12memalloc_update.md)

**요약**: D3D12MemAlloc 라이브러리 업데이트 + Windows 11 최적화
- **버전**: 2.1.0 (2023) → 3.0.1 (2025) - 2년치 업데이트
- **Win11 최적화**: `ALLOCATOR_FLAG_DONT_PREFER_SMALL_BUFFERS_COMMITTED` 플래그
- **문제 해결**: Windows 11에서 작은 committed buffer 성능 저하
- **배경 지식**: Committed vs Placed Resource, D3D12MA 역할

**VizMotive 적용 완료** (2026-01-29)
- `D3D12MemAlloc.h/cpp`: 전체 교체
- `GraphicsDevice_DX12.cpp`: Win11 최적화 플래그 추가

---

### #19. `be8c766e` - fixed critical issue with weak_ptr ✅ 적용 완료
**날짜**: 2025-12-14 | **카테고리**: 버그수정 | **우선순위**: 높음 (Critical)

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/19_be8c766e_weak_ptr_race_condition.md)

**요약**: weak_ptr::lock()의 멀티스레드 race condition 완전 해결
- **문제**: 커밋 #9의 inc→check→dec 방식은 원자적이지 않음, 메모리 재사용 시 다른 객체 영향
- **해결**: `try_inc_refcount()` - CAS로 "refcount가 0이 아니면 원자적으로 증가"
- **부수 효과**: `destruct_on_zero` 파라미터 불필요해져서 제거
- **배경 지식**: Compare-And-Swap (CAS), memory ordering, lock-free 알고리즘

**VizMotive 적용 완료** (2026-01-29)
- `Allocator.h`: try_inc_refcount() 구현, weak_ptr::lock() 수정

---

### #20. `bf2f369b` - dx12 fix: DeleteSubresources didn't handle rtvs and dsvs ✅ 적용 완료
**날짜**: 2025-12-31 | **카테고리**: 버그수정 | **우선순위**: 높음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/20_bf2f369b_delete_subresources_rtv_dsv.md)

**요약**: DeleteSubresources()가 RTV/DSV descriptor를 정리하지 않는 버그 수정
- **문제**: RTV/DSV가 `Texture_DX12`에 있어서 `Resource_DX12::destroy_subresources()`가 접근 못 함
- **증상**: Descriptor leak → 장기 실행 시 heap 고갈
- **해결**: `Texture_DX12`를 `Resource_DX12`에 병합 (vtable 오버헤드 없이 해결)
- **배경 지식**: DX12 Descriptor 종류 (SRV, UAV, RTV, DSV), Subresource 개념

**VizMotive 적용 완료** (2026-01-29)
- `GraphicsDevice_DX12.cpp`: Texture_DX12 클래스 삭제, Resource_DX12에 병합

---

### #21. `3899e472` - Mac OS support ✅ 부분 적용
**날짜**: 2026-01-11 | **카테고리**: 플랫폼 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/21_3899e472_fence_simplification.md)

**요약**: Mac OS + Metal API 지원 (DX12 관련 변경만 적용)
- **Fence 단순화**: 2개 배열(cpu/gpu) → 1개 + monotonic 값 추적
- **RT nullptr 체크**: `WriteTopLevelAccelerationStructureInstance`에 방어 코드 추가
- **스킵**: Metal API 전체, 함수명 리네임
- **배경 지식**: DX12 Fence 동기화 개념

**VizMotive 부분 적용** (2026-01-29)
- `GraphicsDevice_DX12.h/cpp`: Fence 구조 단순화, RT nullptr 체크

---

### #22. `43b93085` - dx12, vulkan: using block allocator for command lists ✅ 적용 완료
**날짜**: 2026-01-15 | **카테고리**: 최적화 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/22_43b93085_block_allocator_commandlist.md)

**요약**: CommandList 메모리 풀링으로 할당 오버헤드 감소
- **변경**: `unique_ptr<CommandList_DX12>` → `BlockAllocator<CommandList_DX12, 64>` 풀링
- **효과**: 할당 10-20배 빠름, 캐시 효율 향상, 단편화 방지
- **VizMotive 추가**: 소멸자에서 명시적 `cmd_allocator.free()` 호출 (WickedEngine에 없음)
- **배경 지식**: BlockAllocator 동작 원리, free_list, placement new

**VizMotive 적용 완료** (2026-01-29)
- `GraphicsDevice_DX12.h`: cmd_allocator + raw pointer 벡터
- `GraphicsDevice_DX12.cpp`: allocate() 사용, 소멸자 정리 코드 추가

---

### #23. `5a067ed2` - console updates and fixes ⏭️ 스킵
**날짜**: 2026-01-17 | **카테고리**: 버그수정 | **우선순위**: 낮음

#### 변경 개요
| 항목 | 내용 |
|------|------|
| 수정 파일 | wiGraphics.h, wiGraphicsDevice_DX12.cpp/h 외 |
| 핵심 변경 | 콘솔(Xbox) 관련 수정 + capability 정리 |

#### 주요 변경 내용

##### 1. `RENDERTARGET_AND_VIEWPORT_ARRAYINDEX_WITHOUT_GS` capability 제거
```cpp
// 이 기능: VS/MS에서 SV_RenderTargetArrayIndex 직접 설정 가능 여부
// 기존: 미지원 GPU를 위해 GS 에뮬레이션 코드 유지
// 변경: 현대 DX12 GPU는 모두 지원하므로 에뮬레이션 코드 삭제
```

WickedEngine은 다음도 함께 삭제:
- `GSTYPE_SHADOW_EMULATION`
- `GSTYPE_ENVMAP_EMULATION`
- `GSTYPE_SHADOW_TRANSPARENT_EMULATION`
- `GSTYPE_SHADOW_ALPHATEST_EMULATION`

##### 2. Enum 플래그 번호 재정렬
제거된 플래그로 인해 `GraphicsDeviceCapability` enum 번호 변경

##### 3. 콘솔(Xbox) 전용 변경
- Xbox용 HEVC GUID 추가
- Xbox swapchain 생성 시 `ApplyTextureCreationFlags` 호출
- Xbox Present/RenderPass 코드 수정

#### 스킵 사유
| 이유 | 설명 |
|------|------|
| 콘솔 전용 | 대부분 Xbox 관련 수정 |
| 호환성 유지 | VizMotive는 GS 에뮬레이션 코드 유지하여 구형 GPU 지원 |
| 기능적 이점 없음 | 코드 정리일 뿐, 성능/기능 개선 없음 |
| 리스크 | enum 재정렬로 인한 호환성 문제 가능 |

---

### #24. `2e823bb2` - region copy support for CPU textures ✅ 적용 완료
**날짜**: 2026-01-18 | **카테고리**: 기능 | **우선순위**: 중간

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/24_2e823bb2_cpu_texture_region_copy.md)

**요약**: CPU 텍스처(UPLOAD/READBACK) 복사 시 footprint 기반 위치 지정 지원
- **문제**: 기존 코드는 모든 경우에 subresource index만 사용 → CPU 텍스처 복사 오류
- **해결**: 4가지 usage 시나리오별로 올바른 copy location 사용
  - UPLOAD→DEFAULT: src=footprint, dst=subresource
  - DEFAULT→READBACK: src=subresource, dst=footprint
  - DEFAULT→DEFAULT: 양쪽 subresource
  - CPU→CPU: 양쪽 footprint
- **헬퍼 함수**: `GetPlaneSlice()`, `ComputeSubresource()` 추가
- **배경 지식**: Footprint 개념, multi-plane 포맷, subresource 인덱싱

**VizMotive 적용 완료** (2026-01-29)
- `GBackend.h`: GetPlaneSlice(), ComputeSubresource() 추가
- `GraphicsDevice_DX12.cpp`: CopyTexture() 4가지 시나리오 처리

---

### #25. `24824a1c` - texture upload improvements ✅ 적용 완료
**날짜**: 2026-01-24 | **카테고리**: 최적화 | **우선순위**: 낮음

#### [상세 분석 문서](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/25_24824a1c_texture_upload_improvements.md)

**요약**: 텍스처 관련 헬퍼 함수 추가 + CreateTexture 코드 정리
- **헬퍼 함수**: `GetMipCount(TextureDesc)`, `GetTextureSubresourceCount()` 추가
- **개선**: mip_levels 계산을 resource 생성 전으로 이동
- **효과**: 코드 중복 제거, 가독성 향상, mip_levels=0 자동 처리
- **스킵**: `CreateTextureSubresourceDatas()` alignment 파라미터 (VizMotive에 함수 없음)

**VizMotive 적용 완료** (2026-01-29)
- `GBackend.h`: GetMipCount(), GetTextureSubresourceCount() 추가
- `GraphicsDevice_DX12.cpp`: mip_levels 계산 시점 변경, 헬퍼 함수 사용

---

## 분석 완료 요약

### 전체 커밋 현황 (25개)

| 상태 | 개수 | 커밋 번호 |
|------|------|-----------|
| ✅ 적용 완료 | 18개 | #1, #6, #7, #8, #9, #10, #11, #12, #14, #15, #16, #17, #18, #19, #20, #21, #22, #24, #25 |
| ✅ 이미 적용됨 | 3개 | #3, #4, #13 |
| ⏭️ 스킵 | 4개 | #2, #5, #23 |

### 주요 적용 내용

#### 버그 수정
- SRV/UAV 벡터 범위 검사 (#1)
- CopyTexture undefined behavior 수정 (#6)
- weak_ptr 이중 소멸자 호출 방지 (#9, #10)
- weak_ptr race condition 해결 (#19)
- DeleteSubresources RTV/DSV 정리 (#20)

#### 최적화
- vtable 제거, 구조체 크기 최적화 (#7)
- pooled shared_ptr (#8, #12)
- CommandList BlockAllocator 풀링 (#22)
- D3D12MemAlloc 3.0.1 업데이트 (#18)

#### 기능 추가
- root descriptor buffer offset 지원 (#16)
- shader-readable swapchain (#17)
- CPU 텍스처 region copy (#24)
- 텍스처 헬퍼 함수 (#25)

#### 코드 품질
- placement new 허용 (#11)
- internal struct final 키워드 (#14)
- clang 경고 수정 (#15)
- Fence 단순화 (#21)

---

## VizMotive 적용 우선순위

(분석 후 정리 예정)

---

## 적용 완료 커밋

| # | 커밋 | 내용 | 적용일 |
|---|------|------|--------|
| 1 | `0bdd7a2a` | SRV/UAV 벡터 범위 검사 | 2026-01-28 |
| 6 | `ccaaf209` | CopyTexture undefined behavior 수정 | 2026-01-28 |
| 7 | `93278bea` | vtable 제거, 구조체 크기 최적화 | 2026-01-28 |
| 8 | `7a5ea2f6` | pooled shared_ptr (DLL 경계 대응 수정) | 2026-01-28 |
| 9 | `e1dc87e4` | weak_ptr::lock() 이중 소멸자 호출 방지 | 2026-01-28 |
| 10 | `16a19429` | destruct_on_zero 기본값 수정 | 2026-01-28 |
| 11 | `90e94ed4` | placement new 허용 | 2026-01-28 |
| 12 | `ffcc3abd` | SharedHeapAllocator + make_shared_single (부분 적용) | 2026-01-28 |
| 14 | `973c2850` | Texture_DX12, BVH_DX12에 final 추가 | 2026-01-28 |
| 15 | `cf6f3434` | clang 경고 수정 (default case) | 2026-01-28 |
| 16 | `f0203679` | root descriptor buffer offset 지원 | 2026-01-28 |
| 17 | `a5162e04` | shader-readable swapchain (SRV 지원) | 2026-01-29 |
| 18 | `7ecbc25a` | D3D12MemAlloc 3.0.1 업데이트 + Win11 최적화 | 2026-01-29 |
| 19 | `be8c766e` | weak_ptr race condition 해결 (try_inc_refcount) | 2026-01-29 |
| 20 | `bf2f369b` | DeleteSubresources RTV/DSV 정리 (Texture_DX12 병합) | 2026-01-29 |
| 21 | `3899e472` | Fence 단순화 + RT nullptr 체크 (부분 적용) | 2026-01-29 |
| 22 | `43b93085` | CommandList BlockAllocator 풀링 | 2026-01-29 |
| 24 | `2e823bb2` | CPU 텍스처 region copy (footprint 지원) | 2026-01-29 |
| 25 | `24824a1c` | 텍스처 헬퍼 함수 + CreateTexture 정리 | 2026-01-29 |

---

## VizMotive 자체 버그 수정

### Image/Font PSO 렌더 타겟 포맷 불일치 수정 (2026-02-02)

#### 문제 상황
Sample08 실행 시 D3D12 에러 발생:
```
D3D12 ERROR: ID3D12CommandList::DrawInstanced: The render target format in slot 0
does not match that specified by the current pipeline state.
(pipeline state = R10G10B10A2_UNORM, render target format = R11G11B10_FLOAT,
RTV ID3D12Resource* = 'rtMain')
```

#### 원인 분석
VizMotive 렌더링 파이프라인에서 두 가지 렌더 타겟 포맷 사용:
| 렌더 타겟 | 포맷 | 용도 |
|-----------|------|------|
| `rtMain` | `R11G11B10_FLOAT` | HDR 내부 렌더링 (3D) |
| `swapChain` / `rtRenderFinal` | `R10G10B10A2_UNORM` | SDR 최종 출력 (2D) |

기존 코드는 Image/Font PSO를 `R10G10B10A2_UNORM` 하나로만 생성하여 `rtMain`에 렌더링할 때 포맷 불일치 발생.

#### 해결책: 포맷별 PSO 생성

##### 1. RT_FORMAT_MODE enum 추가
```cpp
// Image.cpp, Font.cpp
enum RT_FORMAT_MODE
{
    RT_FORMAT_SDR,  // R10G10B10A2_UNORM (swapchain, rtRenderFinal)
    RT_FORMAT_HDR,  // R11G11B10_FLOAT (rtMain)
    RT_FORMAT_MODE_COUNT
};
```

##### 2. PSO 배열 확장
```cpp
// Image.cpp
// 변경 전
static PipelineState imagePSO[2][BLENDMODE_COUNT][...];
static PipelineState debugPSO;

// 변경 후
static PipelineState imagePSO[RT_FORMAT_MODE_COUNT][BLENDMODE_COUNT][...];
static PipelineState debugPSO[RT_FORMAT_MODE_COUNT];

// Font.cpp
// 변경 전
static PipelineState PSO[DEPTH_TEST_MODE_COUNT];

// 변경 후
static PipelineState PSO[RT_FORMAT_MODE_COUNT][DEPTH_TEST_MODE_COUNT];
```

##### 3. LoadShaders에서 두 포맷 모두 PSO 생성
```cpp
// Image.cpp, Font.cpp
Format rtFormats[RT_FORMAT_MODE_COUNT] = {
    Format::R10G10B10A2_UNORM,   // RT_FORMAT_SDR
    FORMAT_rendertargetMain      // RT_FORMAT_HDR (R11G11B10_FLOAT)
};

for (int f = 0; f < RT_FORMAT_MODE_COUNT; ++f)
{
    RenderPassInfo renderpass = {};
    renderpass.rt_formats[0] = rtFormats[f];
    // ... PSO 생성 ...
}
```

##### 4. Params에 HDR 플래그 추가
```cpp
// Image.h
enum FLAGS
{
    // ...
    HDR_RENDER_TARGET = 1 << 16,  // Render target uses R11G11B10_FLOAT format
};

constexpr void enableHdrRenderTarget() { _flags |= HDR_RENDER_TARGET; }
constexpr void disableHdrRenderTarget() { _flags &= ~HDR_RENDER_TARGET; }
constexpr bool isHdrRenderTarget() const { return _flags & HDR_RENDER_TARGET; }

// Font.h - 동일한 패턴
```

##### 5. Draw 함수에서 PSO 선택
```cpp
// Image.cpp
int formatMode = params.isHdrRenderTarget() ? RT_FORMAT_HDR : RT_FORMAT_SDR;
device->BindPipelineState(&imagePSO[formatMode][...], cmd);

// Font.cpp
int formatMode = params.isHdrRenderTarget() ? RT_FORMAT_HDR : RT_FORMAT_SDR;
device->BindPipelineState(&PSO[formatMode][params.isDepthTestEnabled()], cmd);
```

##### 6. 호출부에서 플래그 설정
```cpp
// SpriteDraw_Detail.cpp - DrawSpritesAndFonts (rtMain에 렌더링)

// Sprite 렌더링
params.enableHdrRenderTarget();  // Rendering to rtMain (R11G11B10_FLOAT)

// Font 렌더링
params.enableHdrRenderTarget();  // Rendering to rtMain (R11G11B10_FLOAT)
params.enableDepthTest();        // 3D space font rendering requires depth test
```

#### 수정된 파일 목록

| 파일 | 변경 내용 |
|------|----------|
| Image.h | `HDR_RENDER_TARGET` 플래그 + 헬퍼 함수 추가 |
| Image.cpp | `RT_FORMAT_MODE` enum + PSO 배열 확장 + Draw PSO 선택 로직 |
| Font.h | `HDR_RENDER_TARGET` 플래그 + 헬퍼 함수 추가 |
| Font.cpp | `RT_FORMAT_MODE` enum + PSO 배열 확장 + Draw PSO 선택 로직 |
| SpriteDraw_Detail.cpp | Sprite/Font 렌더링 시 `enableHdrRenderTarget()` + `enableDepthTest()` 호출 |

#### 핵심 포인트
- 2D UI는 swapchain (SDR)에, 3D 스프라이트/폰트는 rtMain (HDR)에 렌더링됨
- 각 컨텍스트에서 올바른 PSO 포맷을 선택해야 D3D12 validation 통과
- 기본값은 SDR이며, HDR 렌더 타겟에 렌더링할 때만 명시적으로 플래그 설정

---

### Sample01/Sample02 종료 시 Hang 수정 (2026-02-02)

#### 문제 상황
Sample01, Sample02 실행 후 창 닫기 시 프로그램이 종료되지 않고 멈춤:
```
[2026-02-02 13:51:13.238] [info] Jobsystem Shutdown...
[2026-02-02 13:51:14.090] [info] GPU buffer suballocated (offset-based): size=42624736, offset=4912
```

"Jobsystem Shutdown..." 이후에도 GPU 버퍼 할당 로그가 출력되며 프로그램이 무한 대기 상태.

#### 원인 분석
Sample01/02에서 비동기 작업(async job)을 시작하고 **종료 전에 대기(wait)하지 않음**:

```cpp
// Sample01 - line 141-160
vz::jobsystem::context ctx_load_obj;
ctx_load_obj.priority = vz::jobsystem::Priority::Low;
vz::jobsystem::Execute(ctx_load_obj, [scene](vz::jobsystem::JobArgs args) {
    vzm::VzActor* root_obj_actor = vzm::LoadModelFile("../Assets/obj_files/skull/...");
    // GPU 버퍼 할당 포함
    scene->AppendChild(root_obj_actor);
});

// ... main loop ...

vzm::DeinitEngineLib();  // ❌ Wait 없이 바로 종료 시도
```

**Race Condition 발생 순서:**
1. 비동기 작업이 모델 파일 로딩 중 (GPU 버퍼 할당 진행 중)
2. `DeinitEngineLib()` 호출 → `jobsystem::ShutDown()` 실행
3. `ShutDown()`이 `alive = false` 설정 후 스레드 종료 대기
4. 비동기 작업이 GPU 버퍼 할당 중 → 종료 시퀀스와 충돌
5. 데드락 또는 무한 대기 발생

#### 해결책
비동기 작업 완료를 기다린 후 엔진 종료:

```cpp
// Sample01
// ... main loop ends ...

// Wait for async jobs to complete before engine shutdown
vz::jobsystem::Wait(ctx_load_obj);

vzm::DeinitEngineLib();

// Sample02 - 동일한 패턴
vz::jobsystem::Wait(ctx_stl_loader);

vzm::DeinitEngineLib();
```

#### 수정된 파일 목록

| 파일 | 변경 내용 |
|------|----------|
| Examples/Sample001/sample01.cpp | `DeinitEngineLib()` 전에 `vz::jobsystem::Wait(ctx_load_obj)` 추가 |
| Examples/Sample002/sample02.cpp | `DeinitEngineLib()` 전에 `vz::jobsystem::Wait(ctx_stl_loader)` 추가 |

#### 핵심 포인트
- 비동기 작업(jobsystem::Execute)을 사용할 때는 종료 전 반드시 Wait() 호출
- `DeinitEngineLib()` 내부의 `jobsystem::ShutDown()`은 대기 중인 작업만 처리
- 실행 중인 작업이 GPU 리소스를 할당하면 race condition 발생 가능
- **규칙**: 앱 종료 전에 모든 async job context에 대해 Wait() 호출 필수
