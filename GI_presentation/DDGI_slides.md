# DDGI 슬라이드 구성
> VizMotive Engine GI Series - Part 1 / Dynamic Diffuse Global Illumination

---

## Slide 1 — 표지
- **제목**: Dynamic Diffuse Global Illumination (DDGI)
- **부제**: VizMotive Engine에 구현한 Real-time GI 시리즈 1편
- 배경: 엔진 실행 스크린샷 (GI 켜진 씬)

---

## Slide 2 — Global Illumination이란?
- 빛은 광원에서 출발해 여러 표면을 **반사·산란**하며 이동
- **직접광(Direct)**: 광원 → 표면 (1번만 이동)
- **간접광(Indirect)**: 광원 → 표면 A → 표면 B → … → 눈
- GI = 직접광 + 간접광을 **모두 계산**하는 것
- 이미지 자리: GI OFF (직접광만) vs GI ON 비교

---

## Slide 3 — 간접광이 씬에 미치는 영향
- **Color bleeding**: 빨간 벽 옆의 흰 벽이 붉게 물드는 현상
- **간접 조명**: 직접 광원이 닿지 않는 곳도 밝아짐 (그림자 속 밝기)
- **Ambient Occlusion**과의 차이: AO는 차폐만, GI는 실제 빛 에너지 이동
- 이미지 자리: Cornell box 씬 Color bleeding 스크린샷

---

## Slide 4 — Ray Tracing이란?

**흔한 오해: "빛에서 ray를 쏜다"**
- 실제 물리: 광원 (셀 수 없는 빛을 방출) → 수조 개의 빛 → 카메라의 정확한 위치를 통과하는 빛은 매우 적음. → 비효율적

**실제 구현: 카메라(눈)에서 각 픽셀 방향으로 ray를 쏜다**

```
카메라
  │
  ├── 픽셀(0,0) 방향으로 ray 발사 → geometry hit → 그 지점의 색 계산
  ├── 픽셀(1,0) 방향으로 ray 발사 → geometry hit → 그 지점의 색 계산
  ├── 픽셀(2,0) 방향으로 ray 발사 → sky hit → 하늘 색
  └── ...
```
- Hit된 지점에서 빛 방향으로 ray 를 발사 -> lighting 또는 shadow 계산 가능
- Hit된 지점에서 **추가 ray를 발사** → 반사/굴절/그림자 계산
- 이 ray가 또 다른 표면에 hit → 다시 ray 발사 → **재귀적 반복**
- 이미지 자리: 카메라 → 픽셀 → ray → hit → bounce 다이어그램

---

## Slide 5 — Path Tracing vs 실시간 GI

**Path Tracing — Ray Tracing을 이용한 GI 계산 방법**
- 각 픽셀에서 여러 개의 ray를 발사하고, 각 ray가 여러 bounce를 거치며 빛의 경로 전체를 추적
- 결과를 Monte Carlo 방식으로 평균내어 수렴 → **수학적으로 가장 정확한 GI**
- 픽사, 디즈니 등 **영화·CG 업계의 표준** (렌더팜에서 프레임당 수십 분~수 시간)

```
1920×1080 해상도, 픽셀당 100 ray 기준:
2,073,600 픽셀 × 100 ray = 약 2억 ray / 프레임
60fps → 초당 약 120억 ray 필요
```

**실시간 GI — 적은 ray로 근사**
- 60fps를 위해 프레임당 수 ms 안에 끝내야 함 → 모든 픽셀에서 수백 개의 ray는 불가
- 핵심 아이디어: **ray를 줄이는 대신 결과를 캐시해두고 재활용**

```
실시간 GI:  수만~수십만 ray / 프레임
            → 대신 여러 프레임에 걸쳐 누적해서 품질 보완
```

| | Path Tracing | 실시간 GI |
|---|---|---|
| 사용처 | 영화, CG 오프라인 렌더링 | 실시간 게임 |
| 프레임당 ray 수 | ~수억 | ~수십만 |
| GI 정확도 | 높음 (ground truth) | 근사 |
| 실시간 가능 여부 | ❌ | ✅ |

---

## Slide 6 — Real-time GI의 해결 방향

**라이팅 정보를 캐시해두고 재활용**

한 프레임에 모든 반사를 계산하는 대신, **여러 프레임에 걸쳐 빛을 전파**

```
Frame 1:  광원 → 표면 A 밝아짐 → [표면 A의 밝기 저장]

Frame 2:  표면 B 라이팅 시 저장된 "표면 A의 빛" 꺼내서 사용
          → 표면 B도 간접적으로 밝아짐 → [표면 B의 밝기 저장]

Frame 3:  표면 A가 이번엔 "표면 B의 빛"까지 받음
          → 빛이 더 멀리까지 전파됨

Frame N:  반복할수록 빛이 씬 전체로 퍼져나감 → 무한 반사 수렴!
```

이 라이팅 정보를 "저장" 방식을 어떻게 구현하느냐에 따라 GI 기법이 나뉨:
- Probe 기반 (DDGI) ← 이번 발표
- Voxel 기반 (VoxelGI)
- Surfel 기반 (SurfelGI)

---

## Slide 7 — DDGI 개요
- **Dynamic Diffuse Global Illumination**
  - Majercik et al., NVIDIA / Morgan McGuire, 2019
- 핵심 아이디어: 씬에 **Irradiance Probe**를 3D 격자로 배치
- 매 프레임 각 Probe에서 **ray를 발사**해 주변 radiance를 캡처
- 캡처된 정보를 **Probe마다 저장** → 셰이딩 시 주변 8개 Probe 보간하여 조회

---

## Slide 8 — Irradiance Probe란?
- 공간의 **한 지점**에 놓인 가상의 센서
- 매 프레임 주변으로 ray를 발사해 **방향별 빛 정보를 수집·갱신**
- 씬이 바뀌어도 실시간으로 따라옴
- 이미지 자리: Probe 구체 디버그 뷰 (엔진에서 구 형태로 시각화된 것)
- 실제 렌더링 시에 표면 근처 probe 에 저장된 값을 이용해 간접광을 계산

---

## Slide 9 — Probe Grid 배치
- 씬 전체를 **3D 격자**로 덮어 Probe를 균일하게 배치
- Probe 간격 = GI 해상도 / 성능 트레이드오프
  - 간격 좁을수록 정밀, 비쌈
- VizMotive 설정값: `[실제 Grid 크기 입력]`
- **Probe Relocation**: Probe가 벽 내부에 박혔을 경우 인접 빈 공간으로 자동 이동
  - VoxelGrid를 이용해 geometry 안에 있는지 판별
- 이미지 자리: Probe Grid 전체 배치 시각화

---

## Slide 10 — 매 프레임 처리 ① Ray 발사 & L_i 수집

**렌더링 방정식 (Rendering Equation)**
```
최종 색 = 자체 발광 + ∫ BRDF × L_i(ω) × cos(θ) dω
                       모든 입사 방향
```
- `L_i(ω)` = 방향 ω에서 들어오는 빛 (incoming radiance)
- Diffuse 표면: BRDF = albedo/π (상수) → 방정식이 단순화됨
- **DDGI가 계산하는 것 = `L_i(ω)` 를 수집해서 적분**

**Probe에서 ray 발사 = L_i(ω) 수집**
- Ray 방향 생성: **Spherical Fibonacci** 수열로 구 표면에 균일하게 분포
- 각 방향 ω로 ray 발사 → hit point에서 `L_i(ω)` 계산:
  - **직접광**: 라이트 방향으로 shadow ray 발사 → 광원에서 오는 빛
  - **간접광**: hit point 주변 Probe에서 간접광 샘플링 ← 무한 반사의 핵심
    - (샘플링 방법 → Slide 13 참조 / 씬 렌더링 셰이딩과 완전히 동일한 함수 사용)
- 수집된 `L_i(ω)` 를 **8×8 텍셀 그리드**에 방향별로 누적 + temporal blending
- 이미지 자리: 8×8 텍셀 맵 시각화 (Probe 하나의 중간 결과)

---

## Slide 11 — 매 프레임 처리 ② L_i(ω) 분포를 SH로 압축 저장

**Probe가 저장하는 것 = 방향별 L_i(ω) 분포**
- 8×8 텍셀로 누적된 방향별 radiance → **SH(Spherical Harmonics)로 압축**
- SH = 구 표면 위 함수를 방향 해상도 단계별로 표현하는 방법
- 이미지 자리: L0 / L1 / L2 SH basis 시각화

```
L0 (1계수)  →  전방향 평균 밝기
L1 (4계수)  →  "어느 쪽이 더 밝은가" 대략적 방향성  ← DDGI 사용 범위
L2 (9계수)  →  더 세밀한 방향 디테일
```

**왜 텍셀이 아닌 SH로 저장하는가?**

| | 8×8 텍셀 | SH L1 |
|---|---|---|
| 저장 크기 | 192 floats | **12 floats** |
| 블렌딩 | 방향별 따로 | 계수 weighted sum (선형) |
| 적분 | 샘플링 반복 필요 | **dot product 한 번** |
| 성격 | 이산적 (64방향) | **연속 함수** |

- 렌더링 시 SH 접근 빈도: 픽셀 수 × 8 probe × 60fps → 초당 수억 회 → 12 floats 캐시 효율이 결정적
- Diffuse L_i 분포는 방향 차이가 완만한 신호 → L1 (12 floats)으로 충분히 근사 가능
- **블렌딩도 O(1), 적분도 O(1)** — SH의 선형 기저 성질 덕분

> Depth(Visibility) 텍스처는 별도 패스에서 Octahedral 텍스처로 저장 (벽 뒤 Probe 기여 차단용)

---

## Slide 12 — GI 적용 — 표면 점 P의 최종 색 계산

**씬 렌더링 시 표면 점 P에서 Probe 간접광 샘플링 → Irradiance 계산**
- 샘플링 방법은 다음 슬라이드(Slide 13)에서 설명
- Probe ray tracing의 간접광 계산(Slide 10)도 동일한 방법 사용

```
① P 주변 Probe 간접광 샘플링 → Irradiance (표면이 받는 빛 합산)

② 최종 색(L_o) = albedo × Irradiance / π
   (Diffuse 표면은 받은 빛을 전방향으로 고르게 반사)
```

---

## Slide 13 — Probe 간접광 샘플링 방법

**표면 점 P, 법선 N 입력 → Irradiance 출력**
- Slide 10 (ray hit 간접광)과 Slide 12 (씬 렌더링) 모두 동일하게 사용

```
[Probe0]──[Probe1]
   │    P    │      ← P가 속한 grid cell의 8개 모서리 probe
[Probe2]──[Probe3]
```

> **직관**: 8개 Probe SH를 위치 가중치로 블렌딩 = 8개 Probe를 하나로 합쳐
> P 위치에 가상의 Probe를 만드는 것. 이 가상 Probe의 SH로 N 반구를 적분.

**계산 순서:**
```
① P 위치 기반으로 8개 Probe SH를 위치 가중치로 블렌딩
   → sum_sh  (P 위치의 가상 Probe L_i 분포)

② sum_sh 를 법선 N 으로 구면 적분
   → Irradiance = ∫ L_i(ω) · cos(θ) dω
   → SH 구면 적분은 계수들의 내적(dot product)으로 계산 가능 → 곱셈 몇 번으로 끝
   → (증명은 구면 조화함수의 직교성 및 회전 불변성에 기반, 상당히 복잡)
```

**Probe 가중치 계산:**
- **Trilinear weight**: P의 grid cell 내 상대 위치 기반
- **Backface test**: 표면 법선 반대편 Probe 기여 감소
- **Visibility test**: Depth 텍스처에서 probe→P 방향 거리 확인 → 벽 뒤 Probe 차단

---

## Slide 14 — 무한 반사 (Infinite Bounces)
- Ray가 hit한 지점에서 **다시 Probe를 조회**
  - Hit point radiance = 직접광 + (이전 프레임 Probe SH에서 가져온 간접광)
- 이 radiance가 Probe SH에 기록 → 다음 프레임 ray hit에서 재활용
- 프레임이 반복될수록 빛이 **계속 전파**됨
- N번 bounce ≈ N 프레임의 시간 지연으로 수렴
- 이미지 자리: 1 bounce → 2 bounce → 수렴 비교

---

## Slide 15 — 전체 흐름 요약 다이어그램
```
[Ray 발사 from Probe]
       ↓
[Hit point 발견]
       ↓
[직접광 (shadow ray)]  +  [이전 프레임 Probe SH 조회 (간접광)]
       ↓
[방향별 8×8 텍셀 누적 + temporal blending]
       ↓
[8×8 텍셀 → L1 SH 프로젝션 → probe.radiance 저장]
       ↓
[셰이딩 시: 주변 8 Probe SH 가중합산 → CalculateIrradiance(sum_sh, N)]
       ↓
[다음 프레임 반복 → 무한 반사 수렴]
```

---

## Slide 16 — VizMotive 구현 결과
- 이미지 자리 ×3~4장:
  - **GI OFF** vs **GI ON** 비교
  - Color bleeding 효과 클로즈업
  - Probe 디버그 뷰 (구체 형태 시각화)
  - 동적 씬 (오브젝트 이동 시 GI 반응)

---

## Slide 17 — DDGI의 장점과 한계
| | 내용 |
|---|---|
| **장점** | 동적 씬 완전 지원, 무한 반사 |
| **장점** | SH 저장으로 Probe당 데이터 매우 컴팩트 (12 floats) |
| **장점** | Raytracing HW 활용, Dominant Light 추출 가능 |
| **한계** | Probe Grid 해상도 한계 → 세밀한 구조 표현 어려움 |
| **한계** | Light leaking (벽 너머 빛 누수) 가능성 |
| **한계** | L1 SH는 고주파 빛 방향성 표현 불가 (specular 부적합) |

---

## Slide 18 — 마무리 & 다음 발표 예고
- DDGI 핵심: **Probe에 매 프레임 ray로 갱신 → L1 SH로 압축 저장 → 무한 반사 수렴**
- 다음: **VoxelGI** — 씬을 Voxel로 나눠 빛 전파를 시뮬레이션하는 방식
