# SurfelGI 디버깅 — 다음 액션 플랜

> 관련: [debugging_log.md](debugging_log.md) §8 결론 — VizMotive D3D12 backend 의 cross-dispatch buffer read 문제로 추정.
>
> 목표: **Wicked Engine 9.2 와 동일 동작 보장** (코드 + backend).

---

## 핵심 가설

같은 `surface.load(prim, bary)` 호출이 Update CS 에서는 정상 (`surface.P` 정상), Binning CS 에서는 NaN/Inf.

→ **VizMotive backend 의 cross-dispatch buffer read** 가 Wicked 와 다르게 동작.
→ 셰이더 코드는 Wicked 와 동일하게 포팅했지만 **backend 차이** 또는 **놓친 셰이더/C++ 차이** 가 원인.

---

## 5단계 계획

### 1단계: 진척도 파악
- **위치**: `C:\graphics\deepdive\Wicked-engine-Deep-Dive\wicked-follow\topics\README.md`
- **목적**: 이전에 진행한 Wicked backend → VizMotive 적용 작업의 진척도 확인. SurfelGI 가 의존할 가능성 있는 **미적용 항목** 식별.
- **출력**: 의심 항목 리스트 (2단계 의존성 매핑에 사용)

### 2단계: SurfelGI 의 backend 의존성 매핑

의심도 높은 순:

| # | 의존성 | 근거 |
|---|--------|------|
| (a) | **`device->Barrier(GPUBarrier::Buffer ..., UAV→SR)` transition** | Update CS 결과를 Binning 에서 read. 우리 진단에서 SR/UAV 모두 stale. |
| (b) | **`DispatchIndirect(buffer, offset)` args offset 처리** | `offsetof(SurfelIndirectArgs, iterate/raytrace/integrate)` 다중 호출. |
| (c) | **`BindUAVs` 다수 슬롯 (Update CS 는 7개 UAV)** | root signature / descriptor table 한도 또는 binding 누락 가능. |
| (d) | **StructuredBuffer view stride** | sizeof(Surfel)=48, sizeof(SurfelData)=32. View stride 가 일관되지 않으면 OOB. |
| (e) | **Cross-queue sync** | Update_SurfelGI (compute queue) 와 SurfelGI_Coverage (graphics queue) 사이 fence. |
| (f) | **`CreateBufferZeroed` / `CreateBuffer(initial_data)`** | deadBuffer 역순 초기화, statsBuffer 초기 stats. 첫 프레임에 garbage 일 가능성. |

### 3단계: 의존성별 문서 검색 + 미적용 patch

각 (a)~(f) 항목에 대해:

1. `wicked-follow/topics/` 에서 관련 문서 검색 (e.g. "Barrier", "DispatchIndirect", "BindUAV" 등)
2. **적용된 항목**: VizMotive `GraphicsDevice_DX12.cpp` 와 Wicked `wiGraphicsDevice_DX12.cpp` 의 해당 함수 1:1 비교. 차이 있으면 patch.
3. **미적용 항목**: Wicked 코드 가져와 적용. 단, 다른 기능에 영향 가는지 확인.

### 4단계 (병행): SurfelGI 셰이더 + C++ 코드 1:1 diff 재검토

진단 결과 Update CS 까지는 정상 (cell 200 = 0, surface.P 정상). Binning 부터 깨짐. 따라서 다음 코드들을 정밀 재비교:

- `surfel_binningCS.hlsl` ↔ Wicked
- `surfel_coverageCS.hlsl` ↔ Wicked
- `surfel_integrateCS.hlsl` ↔ Wicked (특히 SH::Pack/Unpack 호출)
- `surfel_raytraceCS.hlsl` ↔ Wicked
- `GI_Detail.cpp::Update_SurfelGI()` ↔ Wicked `SurfelGI()`
- `GI_Detail.cpp::SurfelGI_Coverage()` ↔ Wicked `SurfelGI_Coverage()`

확인 항목:
- Binding 순서 (t/u 슬롯)
- Barrier 종류 / 호출 시점
- Push constant
- DispatchIndirect offset 값

### 5단계: 발견된 fix 적용 후 검증

진단 코드는 유지한 채 빌드 후 4분할 결과 변화 확인:

| 진단 위치 | 정상 변화 (fix 효과) |
|---------|------------------|
| 좌상 R (uid==0) | **빨강 → 검정** |
| 좌하 R (실패) → G (성공) | G 만 강하게 saturate |
| 우하 R (NaN/Inf) | **빨강 → 검정** |

정상화 확인되면:
- 진단 코드 모두 제거
- 정식 디버그 모드 (Heatmap/Random/Color/Normal/Inconsistency/Point) 검증
- `debugging_log.md` 의 § 8 결론 업데이트

---

## 추가 옵션 (필요 시)

- **PIX 캡처**: GPU memory 직접 검증. sample14 빌드에서 `--pix` 같은 옵션 활성화 시도.
- **D3D12 GPU-based validation**: 더 깊은 hazard 검출.
- **Dxc reflection 으로 stride 직접 확인**: `dxc -reflection` 으로 surfel struct 의 실제 offset 확인.

---

## 작업 순서 권장

1. **1단계 → 2단계**: 사전 분석 (변경 없음). 시간 짧음.
2. **3단계**: 변경 가장 클 가능성. 각 의존성 항목 별로 진행.
3. **4단계 병행**: 3단계 진행 중 셰이더 diff 도 동시에. 빠른 발견 가능.
4. **5단계**: 변경 적용 후 검증.

---

## 메모: 임시 진단 코드 현재 상태

- `surfel_updateCS.hlsl`: cell 200/201/202/203 진단 (surface.P NaN, length>1k, ==0, 성공 카운트). **검증 결과 cell 200 = 0 (Update 정상)**, 이미 검증 완료. 제거 가능.
- `surfel_binningCS.hlsl`: cell 100/101/102/103 (dispatch/GetRadius/cellintersects/NaN), cell 210~215 (uid==0 등 필드 검증). **유지 — 5단계 검증에 사용**.
- `surfel_coverageCS.hlsl`: 4분할 진단 (좌상 uid==0/primID==0, 우상 bary==0/maxinc==0, 좌하 load 실패/성공, 우하 NaN/Inf). **유지**.
- `GI_Detail.cpp::SurfelGI_Coverage()`: `dataBuffer` 추가 binding, **Plan A 의 Binning 변경 (UAV view, global memory barrier) 유지**.

5단계 검증 후 모두 원복 또는 정리.
