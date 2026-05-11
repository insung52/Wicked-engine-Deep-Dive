# VizMotive Surfel GI 디버깅 로그

> 진행 중 (Phase 5 검증 단계). Wicked 9.2 Surfel GI 를 VizMotive Engine 의 sample14 에 통합.
>
> 관련 문서:
> - 분석: [surfelGI.md](surfelGI.md)
> - 통합 계획: [vizmotive_surfelgi_plan.md](vizmotive_surfelgi_plan.md)

---

## 1. 핵심 증상

SurfelGI 활성화 + 디버그 모드 (Heatmap/Color/Random/Normal/Point/Inconsistency) 켜도 화면에 거의 변화 없음:

- **Color, Heatmap**: 화면 고정 위치에 작은 점만 보임 (메시 표면 따라가지 않음)
- **Normal/Random/Point/Inconsistency**: 메시 라이팅 색상 그대로 — 디버그 시각화 안 보임

→ **Surfel 들이 grid 에 거의 등록 안 됨** (`cell.count = 0` for mesh surface pixels)

---

## 2. 진행 상황 (Phase 1~4 완료)

| Phase | 상태 | 내용 |
|-------|------|------|
| Phase 1 인프라 | ✅ | ShaderInterop_SurfelGI.h, GSceneDetails::SurfelGI, GPU 리소스 생성, isSurfelGIEnabled 플래그, RenderPath3D resources, Postprocess_Upsample_Bilateral 래퍼 |
| Phase 2 셰이더 | ✅ | 8개 surfel CS 셰이더 포팅 (coverage/indirectprepare/update/gridoffsets/binning/raytrace[/_rtapi]/integrate), CSTYPE 등록, vcxitems/compile.bat |
| Phase 3 파이프라인 | ✅ | Update_SurfelGI() (compute queue), SurfelGI_Coverage() (graphics queue), Render() flow hookup |
| Phase 4 Sample14 UI | ✅ | SurfelGI checkbox + Debug mode combo, SURFELGI_ENABLED config |
| **Phase 5 검증** | 🔴 **진행 중** | 셰이더 dispatch 흐름 정상이지만 mesh 표면에 cell.count = 0 |

---

## 3. 진단 결과 (Phase 5)

### 3.1 검증된 정상 동작 ✅

| 검증 항목 | 결과 |
|---------|------|
| 셰이더 8개 모두 .cso 컴파일 | ✅ 출력 폴더에 7개 (RT 지원 GPU 라 SW BVH 빠진 1개 정상) |
| Coverage CS dispatch | ✅ 화면 강제 빨간색 디버그 코드로 검증 |
| Coverage CS early return 없음 | ✅ depth/primitiveID/surface.load 모두 통과 (mesh 표면에 흰색) |
| Update CS dispatch | ✅ 강제 진단 InterlockedAdd 작동 |
| Update CS InterlockedAdd 자체 | ✅ 같은 셰이더 내 다른 InterlockedAdd 정상 |
| Update CS 의 surface.load 분기 진입 | ✅ cell 1, 2, 3 진단 성공 |
| Update CS 의 27 cells 등록 | ✅ cellAllocator > 50000 (alive × 27 누적) |
| Indirect args | ✅ iterate, integrate saturate, raytrace 시간 따라 감소 (정상) |
| Stats (count, nextCount, alive) | ✅ 모두 saturate (alive 1000+) |
| Binning CS dispatch | ✅ cell 100 진단 성공 |
| Binning CS GetRadius>0 분기 | ✅ cell 101 진단 성공 |

### 3.2 핵심 미해결 문제 ❌

| 문제 | 증거 |
|------|------|
| **Binning 시점의 surfel.position 이 NaN/Inf** | cell 103 진단 (isnan/isinf 검사) 카운트 증가 |
| **Binning 의 surfel_cellintersects 가 항상 false** | cell 102 진단 = 0 |
| **Mesh 표면의 cell.count = 0** | pixel cell 의 surfelGridBuffer[ci].count = 0 |
| **Update CS 와 Binning CS 의 surfel.position 이 다른 값** | Update CS 시점엔 cellAllocator > 50000 (정상), Binning 시점엔 NaN/Inf |

→ **같은 surfelBuffer 의 같은 surfel_index 를 두 다른 dispatch 가 read 인데 다른 결과**.

### 3.3 시도했으나 효과 없는 fix

| 시도 | 효과 |
|------|------|
| `uid_validate` 를 `uint64_t` 로 (VizMotive Entity 가 64bit) | uid 검증은 통과 (이게 진짜 fix 였음) — 하지만 cell.count 0 문제는 미해결 |
| primitiveID 직접 binding (visibility_resolveCS 패턴) | primitiveID 정상 read — 해결책 아님 |
| Update→GridOffsets 사이 GPUBarrier::Memory 추가 | cellAllocator 는 정상이지만 cell.count 0 여전 |
| GridOffsets→Binning 사이 GPUBarrier::Memory 추가 | 효과 없음 |
| `isAccelerationStructureUpdateRequested` 게이트에 `isSurfelGIEnabled` 추가 | TLAS 빌드 요청 정상 (필수 fix) |
| Lazy create surfelGIResources in Render() | runtime 토글 시 result 텍스처 정상 생성 (필수 fix) |
| **Surfel layout 변경 #1**: `SH::L1_RGB::Packed radiance` (원래) | NaN/Inf 발생 |
| **Surfel layout 변경 #2**: `uint radiance[6]` (array) | **여전히 NaN/Inf** |
| **Surfel layout 변경 #3**: `uint radiance0~5` (개별) | **여전히 NaN/Inf** |

→ **Layout 가설은 틀림**. 다른 근본 원인 탐색 필요.

---

## 4. 다음 계획 (B → A → C 순서)

### Plan B: Update CS 의 surface.P 가 NaN/Inf 인지 검증 ⭐ 1순위

목적: surface.load 의 결과 자체가 깨졌는지, 아니면 surfelBuffer write/read 사이 sync 문제인지 분리.

방법:
- Update CS 의 `surface.load(prim, bary)` 호출 후 `if (any(isnan(surface.P)))` 검사 카운터 추가
- 별도 cell (예: cell 200) 에 InterlockedAdd
- Coverage CS 가 그 cell.count 를 시각화

결과 해석:
- cell 200 > 0 → **surface.P 자체가 NaN** (surface.load 또는 attribute_at_bary 가 잘못된 값 반환). 원인을 surfaceHF 또는 vertex buffer 로 좁힘.
- cell 200 = 0 → surface.P 정상. **NaN/Inf 는 surfelBuffer write/read 사이에 발생** (cross-dispatch sync 문제). Plan A 또는 C 로.

### Plan A: Binning CS 가 surfelBuffer 의존 우회 ⭐ 2순위

목적: surfelBuffer cross-dispatch sync 문제 우회. 임시 fix 로 작동 여부 검증.

방법:
- Binning CS 에서 `surfel.position = surfelBuffer[idx].position` 대신 surfelDataBuffer 에서 prim+bary read 후 surface.load 직접 호출
- Update CS 와 동일한 작업 반복 (비효율). 하지만 surfelBuffer 의 stale read 우회.

결과 해석:
- mesh 표면에 cell.count > 0, 디버그 모드 정상 작동 → **surfelBuffer cross-dispatch sync** 가 진짜 원인
- 여전히 cell.count = 0 → 다른 원인. Plan C 로.

성공 시 후속:
- 임시 fix 유지 (Wicked 와 다른 방식이지만 작동)
- 또는 backend transition 디버그 (Plan C 와 병행)

### Plan C: VizMotive D3D12 backend 의 transition/view 처리 검증 ⭐ 3순위

목적: Wicked 와 우리 backend 의 차이 분석. UAV/SR view 의 stride 처리, BindResource/BindUAVs 의 root signature 영향 등.

검증 항목:
1. `GraphicsDevice_DX12::CreateBuffer` 의 stride 계산
2. SR view 와 UAV view 가 같은 stride 사용하는지
3. `Barrier(GPUBarrier::Buffer(buf, UAV→SR))` 의 실제 D3D12 ResourceBarrier 호출
4. `BindResource(t)` 와 `BindUAVs(u)` 가 같은 root signature 안의 dispatch 인지
5. 같은 buffer 의 SR + UAV 동시 binding 시 처리

검증 도구:
- PIX 캡처
- D3D12 debug layer 출력 (이미 활성화됨, 에러 없음 — 따라서 spec 위반은 아님)
- Wicked 의 `wiGraphicsDevice_DX12.cpp` vs VizMotive `GraphicsDevice_DX12.cpp` diff

---

## 5. 현재 코드 상태

### 5.1 임시 진단 코드 (운영 코드와 분리됨)

**제거 필요 (검증 완료 후):**
- `surfel_coverageCS.hlsl` 끝부분의 4분할 진단 (현재: cellAllocator/cell102/cell103/pixel cell.count 시각화)
- `surfel_binningCS.hlsl` 의 NaN/Inf 검사 (cell 103 InterlockedAdd)
- `surfel_coverageCS.hlsl` 의 진단용 t6/t7 binding (없음, 이전에 제거됨)

**유지 (원래 코드):**
- 기타 모든 surfel CS 셰이더는 Wicked 9.2 와 일치

### 5.2 정식 코드 변경 (유지)

이 변경들은 VizMotive 환경에 맞춘 필수 수정:

| 파일 | 변경 |
|------|------|
| `surfaceHF.hlsli:90` | `uint uid_validate` → `uint64_t uid_validate` (Entity = uint64_t) |
| `ShaderInterop_SurfelGI.h` Surfel 구조 | radiance 를 `uint radiance0~5` 개별 필드로 (다만 layout fix 효과 없으므로 원복 가능) |
| `Renderer.cpp::Render()` | Lazy create surfelGIResources |
| `SceneUpdate_Detail.cpp` | `isAccelerationStructureUpdateRequested` 게이트에 `isSurfelGIEnabled` 추가 |
| `GI_Detail.cpp::Update_SurfelGI()` | Update→GridOffsets 사이 `GPUBarrier::Memory(gridBuffer)` 추가 (효과 미확인) |
| `surfel_coverageCS.hlsl` | primitiveID 를 `bindless_textures_uint` 대신 `Texture2D<uint>` t4/t5 직접 binding (visibility_resolveCS 패턴) |

### 5.3 Surfel struct 현재 상태

```cpp
struct alignas(16) Surfel {
    uint radiance0;  // SH L1 RGB packed (각각 half2 packed)
    uint radiance1;
    uint radiance2;
    uint radiance3;
    uint radiance4;
    uint radiance5;
    uint2 normal;        // packed half3
    float3 position;
    uint padding1;
    
    // HLSL only:
    inline float GetRadius() { return SURFEL_MAX_RADIUS; }
    inline SH::L1_RGB UnpackRadiance() { ... }
    inline void PackRadiance(SH::L1_RGB sh) { ... }
};
```

C++ sizeof = 48 bytes. HLSL 도 동일해야 함 (가정).

---

## 6. 디버깅 메모

### 디버그 시각화 패턴 분석

화면 4분할 진단 형태로 한 화면에 여러 metric 시각화. 사용자가 디버그 모드를 None 외 값으로 설정해야 `isSurfelGIDebugEnabled = true` → debugUAV 가 화면에 합성됨.

```hlsl
write_debug(DTid.xy, debug_value);  // 2x2 fullres 픽셀에 같은 색
```

### "화면 고정 점" 현상 해명

Coverage CS 의 spawn 코드에서 `if (deadCount <= 0) return;` 또는 `if (rng < chance) return;` 시 그 픽셀의 `write_debug` 호출 안 됨. 그 픽셀의 debugUAV 는 클리어 안 된 상태 → BLEND_PREMULTIPLIED 합성 시 alpha=0 → rtMain (메시 라이팅) 통과.

매 프레임 minGTid (그룹당 1픽셀) 가 RNG 기반이라 균일 간격 패턴.

### 사용자 환경

- Add-SurfelGI 브랜치 (Add-VoxelGI 기반)
- GPU validation 활성화, 에러 없음
- RT 지원 GPU (RTAPI shader 사용 중)
- sample14 의 cube 테스트 씬

---

## 7. 시간 순 기록

1. ✅ Phase 1~4 구현 완료, 빌드 성공
2. ❌ SurfelGI 켜도 화면 변화 거의 없음 → cell.count 0 의심
3. 진단 단계 1: 셰이더 dispatch 자체 검증 (강제 빨간색) → 정상
4. 진단 단계 2: early return 단계별 색상 → mesh 표면 모두 통과 (흰색)
5. 진단 단계 3: cell.count 시각화 → mesh 표면 모두 검정 (count=0)
6. 진단 단계 4: cellAllocator/stats 시각화 → stats 정상, cellAllocator > 50000
7. 진단 단계 5: surfel.position 시각화 → 처음 결과 모순 (검정), 재시도 시 색상
8. 진단 단계 6: cellAllocator 단계 진단 → 빨강→노랑→초록→시안 (정상 진행)
9. 진단 단계 7: Binning 단계별 (cell 100/101/102) → cell 102 = 0 (cellintersects 모두 false)
10. 진단 단계 8: Binning 의 surfel.position NaN/Inf 검사 (cell 103) → **빨강 = NaN/Inf 확정**
11. Layout fix 시도 #1, #2, #3 → 모두 효과 없음 (NaN/Inf 여전)
12. Plan B: Update CS 의 surface.P 검증 → **정상 (cell 200 = 0)**, Binning 만 NaN
13. Plan A: Binning 에서 surface.load 직접 호출 (surfelBuffer.position 우회) → **여전 NaN/Inf**
14. SR view 우회 (UAV 만 사용) → 효과 없음
15. Global memory barrier (`GPUBarrier::Memory()`) → 효과 없음
16. surfel_data 필드별 검증 → **uid==0 일부, 다른 필드 (primitiveID/bary/max_inconsistency) 정상**
17. uid 를 uint64_t → uint 변경 → 효과 없음 (uid 여전 일부 0)
18. **결론**: surface.load 통과 후에도 surface.P NaN/Inf — Binning 의 surface.load 가 호출하는 vertex buffer / instance transform read 자체가 cross-dispatch 시 garbage. **VizMotive backend 의 cross-dispatch buffer read 이슈**.

## 8. 최종 결론 및 다음 단계

### 진짜 원인
**VizMotive 의 D3D12 backend 가 cross-dispatch (Update CS → Binning CS) buffer read 시 garbage 반환.**

증거:
- 같은 `surface.load(prim, bary)` 호출인데 Update CS 정상, Binning CS NaN
- surfelBuffer/dataBuffer/instance buffer/vertex buffer 모두 영향
- UAV view, SR view, global memory barrier 모두 효과 없음
- Wicked Engine 의 동일 셰이더 코드는 작동 (그쪽 backend 와 차이)

### 가능 해결 방향
1. **Plan C — Backend 분석/패치**: `GraphicsDevice_DX12.cpp` 의 ResourceBarrier / CreateBuffer / view 생성 / descriptor heap 처리를 Wicked 의 `wiGraphicsDevice_DX12.cpp` 와 1:1 비교. 시간 많이 듬.
2. **임시 큰 우회**: Update CS 가 grid 등록 + cellBuffer 채우기까지 모두 수행. Binning CS 제거. cellBuffer offset 을 다음 프레임으로 미룸. 큰 변경.
3. **SurfelGI 작업 보류**: 다른 영역 작업 후 backend 시간 투자.
