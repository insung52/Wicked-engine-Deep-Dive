# SurfelGI 슬라이드 구성
> VizMotive Engine GI Series - Part 2 / Surfel-based Global Illumination

---

## Slide 1 — 표지
- **제목**: Surfel-based Global Illumination (SurfelGI)
- **부제**: VizMotive Engine GI 시리즈 2편
- 배경: 엔진 실행 스크린샷 (SurfelGI 켜진 씬)

---

## Slide 2 — SurfelGI란? (DDGI와 비교)

**DDGI와 SurfelGI는 같은 목표: Diffuse 간접광을 캐시해서 재활용**

| | DDGI | SurfelGI |
|---|---|---|
| 캐시 단위 | Irradiance Probe | **Surfel** |
| 캐시 배치 | 3D grid (고정) | **표면 위 동적 배치** |
| 저장 형식 | L1 SH (12 floats) | L1 SH (12 floats) |
| Specular | ❌ | ❌ |
| 무한 반사 | ✅ | ✅ |
| 무한 세계 | 제한적 | ✅ (hashing) |

> 핵심 차이: **어디에** 캐시를 배치하는가

---

## Slide 3 — Surfel이란?

**Surfel (Surface Element)**: 씬 표면 위에 동적으로 배치되는 작은 원형 디스크

```
                    ● Surfel (표면 위에 부착)
                   /
   씬 geometry ───/────────────
                 ↑
         반경 r 의 원형 영역을 커버
```

- **DDGI probe**: 씬과 무관한 고정 격자 위에 배치
- **Surfel**: 실제 표면 위에 부착 → geometry가 있는 곳에만 존재
- 표면의 primitiveID + barycentric 좌표로 위치 추적
  - **동적 오브젝트가 움직여도 surfel이 따라감**
- 이미지 자리: Surfel 디버그 뷰 (표면 위 점들 시각화)

---

## Slide 4 — DDGI와의 공통점

**저장·적분 방식이 거의 동일**

```
[DDGI]
  ray 결과 → 8×8 texel 누적 → MultiscaleMeanEstimator → L1 SH 저장

[SurfelGI]
  ray 결과 → 4×4 texel 누적 → MultiscaleMeanEstimator → L1 SH 저장
```

공통점 목록:
- **L1 SH (12 floats)** 로 radiance 저장 및 조회
- **텍셀 단위 temporal blending** (MultiscaleMeanEstimator)
- **Inconsistency 기반 ray budget 배분** — 변화가 큰 영역에 ray를 더 많이
- **Visibility test** (Moment texture, Chebyshev 부등식)
- **Shadow ray**로 직접광 계산 (NEE)
- **이전 프레임 데이터 재귀 조회** → 무한 반사

---

## Slide 5 — 핵심 차이: Coverage 기반 동적 배치

**DDGI**: probe는 항상 grid에 고정 → geometry 없는 빈 공간에도 probe 존재

**SurfelGI**: 픽셀 커버리지를 기준으로 필요한 곳에만 surfel 생성

```
Coverage = 픽셀에 영향을 주는 surfel 수 (가중치 포함)

Coverage < 목표치 (0.8)  →  새 surfel 스폰
Coverage ≥ 목표치       →  이미 충분히 커버됨, 스폰 안 함
```

- Surfel이 적은 곳(빈 영역, 새로 등장한 표면) → 자동으로 surfel 추가
- 카메라 frustum 밖으로 나간 surfel → 일정 시간 후 recycle (재활용)
- **최대 100,000개 surfel pool** — dead list / alive list로 관리

---

## Slide 6 — Surfel 생명주기

```
[스폰]
  Coverage 부족 픽셀 감지
  → Dead list에서 인덱스 꺼내기 (재활용)
  → 표면 위치(primitiveID, barycentric) 기록
  → Alive list에 추가
         ↓
[누적]
  매 프레임 ray 발사 → radiance 축적
  Life 카운터 증가 (0~255)
  Inconsistency 감소 → ray 수 줄어듦 (수렴)
         ↓
[재활용]
  카메라 frustum 밖 → Recycle 카운터 증가
  일정 시간 경과 + surfel 부족 상황 → Dead list로 반환
  → 다른 곳에 재스폰
```

---

## Slide 7 — 전체 파이프라인

매 프레임 실행 순서:

```
① Binning CS
   살아있는 surfel → Hash grid cell에 등록

② Grid Offsets CS
   Hash grid cell offset 계산 (조회용)

③ Coverage CS
   픽셀별 주변 surfel 조회 → coverage 계산
   coverage 부족 픽셀 → 새 surfel 스폰
   → GI 결과 출력 (이전 프레임 surfel SH 사용)

④ Indirect Prepare CS
   살아있는 surfel 수 기반 dispatch args 준비

⑤ Update CS
   surfel 위치 업데이트 (표면 추적)
   ray count 결정 (inconsistency 기반)
   recycle 처리

⑥ Raytrace CS
   각 surfel에서 ray 발사
   직접광 (shadow ray) + 간접광 (이전 프레임 surfel 조회)

⑦ Integrate CS
   ray 결과 → 4×4 texel 누적 → temporal blend → L1 SH → surfel.radiance 저장
```

---

## Slide 8 — ⑦ Integrate CS (DDGI와 비교)

**DDGI Update CS와 구조가 동일**

```
[DDGI]
  8×8 texel에 방향별 ray 가중 누적
        ↓
  MultiscaleMeanEstimator (temporal blend)
        ↓
  8×8 → L1 SH 프로젝션 → probe.radiance 저장

[SurfelGI]
  4×4 texel에 방향별 ray 가중 누적
        ↓
  MultiscaleMeanEstimator (temporal blend)
        ↓
  4×4 → L1 SH 프로젝션 → surfel.radiance 저장
```

차이:
- 텍셀 크기: 8×8 → **4×4** (surfel이 더 작은 범위를 커버)
- 방향 기준: 구 전체 → **법선 기준 반구** (hemisphere)
- Moment texture: 16×16 (probe) → **4×4 (surfel)**

---

## Slide 9 — 무한 반사 & 무한 세계

**무한 반사 (DDGI와 동일한 원리)**
- Raytrace CS에서 ray hit point 주변의 surfel 조회
- 이전 프레임 surfel SH에서 간접광 샘플링
- 이 값이 surfel에 기록 → 다음 프레임 재활용 → 무한 반사 수렴

**무한 세계 지원 (DDGI와의 차이)**
- DDGI: Grid 범위 내에서만 동작
- SurfelGI: **Hash grid** 사용 → 위치를 해시값으로 변환
  - Grid 크기 제약 없이 어느 위치든 surfel 등록/조회 가능
  - 카메라가 멀리 이동해도 자연스럽게 대응

---

## Slide 10 — Ray Budget 배분

**전체 ray 예산을 살아있는 surfel들이 공유**

```
SURFEL_RAY_BUDGET = 500,000 ray / 프레임
SURFEL_RAY_BOOST_MAX = 64 ray / surfel (최대)
```

각 surfel의 ray 수 = **inconsistency × 최대 ray 수**
- 처음 스폰된 surfel: inconsistency 높음 → ray 많이
- 수렴된 안정적 surfel: inconsistency 낮음 → ray 거의 없음
- Camera frustum 밖 surfel: ray 감소 → recycle 준비

→ ray를 필요한 곳에 집중해서 효율적으로 사용

---

## Slide 11 — VizMotive 구현 결과
- 이미지 자리 ×3~4장:
  - **GI OFF** vs **GI ON** 비교
  - Color bleeding 효과
  - Surfel 디버그 뷰 (점들이 표면 위에 분포하는 모습)
  - 동적 오브젝트 이동 시 surfel 추적

---

## Slide 12 — SurfelGI의 장점과 한계

| | 내용 |
|---|---|
| **장점** | 표면이 있는 곳에만 배치 → 효율적 |
| **장점** | 동적 오브젝트 추적 (표면에 부착) |
| **장점** | Hash grid로 무한 세계 지원 |
| **장점** | Ray budget을 필요한 surfel에 집중 |
| **한계** | Surfel 스폰/소멸 관리 복잡 |
| **한계** | Surfel 반경(2 world unit)보다 작은 세부 구조에서 GI 정확도 저하 및 light leaking 가능 |
| **한계** | Coverage가 부족한 영역(새로 등장한 표면 등)은 첫 몇 프레임간 GI 품질 저하 |
| **한계** | Diffuse 전용 (L1 SH, Specular 불가) |

---

## Slide 13 — 세 기법 최종 비교

| | DDGI | SurfelGI | VoxelGI |
|---|---|---|---|
| **캐시 단위** | Grid probe | 표면 surfel | 3D voxel |
| **배치** | 고정 격자 | 동적 (표면 위) | 고정 격자 |
| **저장 형식** | L1 SH | L1 SH | Anisotropic RGBA |
| **샘플링** | SH 내적 | SH 내적 | Cone tracing |
| **Specular** | ❌ | ❌ | ✅ |
| **무한 세계** | 제한적 | ✅ | Clipmap |
| **동적 오브젝트** | ✅ | ✅ (표면 추적) | ✅ |
| **Ray tracing 필요** | ✅ | ✅ | ❌ |

---

## Slide 14 — 마무리
- SurfelGI 핵심: **DDGI와 같은 L1 SH 방식 + 표면 위 동적 배치**
- 세 기법은 각각의 장단점이 있으며 씬 특성에 따라 선택
- GI 시리즈 완료: DDGI → SurfelGI → VoxelGI
