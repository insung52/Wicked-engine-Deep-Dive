# VoxelGI 슬라이드 구성
> VizMotive Engine GI Series - Part 2 / Voxel-based Global Illumination

---

## Slide 1 — 표지
- **제목**: Voxel-based Global Illumination (VoxelGI)
- **부제**: VizMotive Engine GI 시리즈 2편
- 배경: 엔진 실행 스크린샷 (VoxelGI 켜진 씬)

---

## Slide 2 — Voxel이란?
- **Voxel (Volumetric Pixel)**: 3D 공간을 격자(grid)로 나눈 단위 요소
  - 2D 이미지의 픽셀처럼, 3D 공간의 각 격자 칸 하나
- 각 voxel에 색상, 법선, 직접광 등의 정보를 저장
- 이미지 자리: voxel grid 시각화 (격자로 나뉜 3D 공간)
- 예시: MRI는 인체를 voxel로 스캔해 3D 구조 재현

---

## Slide 3 — VoxelGI 개요
- **핵심 아이디어**: 씬 전체를 voxel grid로 변환 → 각 voxel에 빛 정보를 저장
- 저장된 voxel 데이터를 **Cone Tracing**으로 샘플링 → 간접광 계산
- DDGI와의 차이:
  - DDGI: 공간 내 고정 probe에서 ray tracing
  - VoxelGI: 씬 전체를 voxel로 표현 → cone tracing으로 샘플링

**VoxelGI가 DDGI보다 추가로 지원하는 것: Specular GI**
- Cone 방향/각도를 자유롭게 설정 가능
- 반사 방향으로 좁은 cone → specular 근사 가능

---

## Slide 4 — 전체 파이프라인 개요

매 프레임 실행 순서:
```
① Voxelization      — 씬 geometry → voxel grid (직접광 포함)
        ↓
② Temporal Processing — voxel에 간접광 계산 + temporal blending
        ↓
③ SDF Jump Flood    — 빈 voxel의 표면까지 거리 계산
        ↓
④ Resolve Diffuse   — 화면 픽셀에서 diffuse GI 샘플링
        ↓
⑤ Resolve Specular  — 화면 픽셀에서 specular GI 샘플링
```

> 최적화: 매 프레임 모든 clipmap을 업데이트하지 않고, 하나씩 로테이션으로 업데이트

---

## Slide 5 — Clipmap 구조

**문제**: 씬 전체를 단일 해상도 voxel로 표현하면 카메라 근처는 너무 거칠고, 멀리는 낭비

**해결**: **Clipmap** — 여러 해상도 레벨로 공간을 중첩 표현

```
레벨 0 (가장 세밀)  — 카메라 근처, 작은 voxel 크기
레벨 1             — 더 넓은 범위, voxel 크기 2배
레벨 2             — 더 넓은 범위, voxel 크기 4배
...
레벨 5 (가장 거침)  — 넓은 범위 커버, voxel 크기 32배
```

- 각 레벨은 동일한 해상도(64³)의 격자
- 카메라 이동 시 clipmap 중심이 카메라를 따라 이동 (양자화된 이동)
- Cone Tracing 시 거리에 따라 적절한 clipmap 레벨 자동 선택 (LOD)
- 이미지 자리: Clipmap 레벨별 커버 범위 시각화

---

## Slide 6 — ① Voxelization

**씬의 모든 geometry를 3D voxel grid로 변환**

**작동 방식** (VS → GS → PS 파이프라인):

**Vertex / Geometry Shader:**
- 각 삼각형의 법선을 분석 → **지배적 축** 결정 (X/Y/Z 중 삼각형이 가장 잘 보이는 방향)
- 해당 축으로 삼각형을 투영 → 가상의 64×64 화면에 래스터화
- 더 많은 voxel이 커버되도록 최적 투영 방향 선택

**Pixel Shader:**
- 래스터화된 각 voxel에서 실행
- 직접광 계산 (ForwardLighting): 표면 위치 P, 법선 N으로 광원들의 기여 합산
- voxel에 저장하는 데이터 (6방향 × 13채널):
  - Base Color (RGBA)
  - Emissive
  - Direct Light
  - Surface Normal
  - Fragment Counter (평균 계산용)
- InterlockedAdd로 원자적 기록 (여러 삼각형이 동시에 같은 voxel에 기록 가능)

- 이미지 자리: Voxelization 결과 디버그 뷰

---

## Slide 7 — Voxel의 방향성 (Anisotropic Storage)

**왜 6방향으로 저장하는가?**

하나의 voxel에 하나의 색상만 저장하면 방향 정보가 사라짐:
- 벽의 앞면 → 밝음 / 뒷면 → 어두움 → 같은 voxel인데 값이 달라야 함

**해결: Anisotropic voxel** — 6개 면 방향(±X, ±Y, ±Z)으로 별도 저장

```
각 voxel = 6개 면 방향 데이터
  +X 면: 이 방향에서 바라볼 때의 색/조명
  -X 면: 반대 방향
  +Y, -Y, +Z, -Z 면도 동일
```

- Cone Tracing 시 진행 방향에 맞는 3개 면을 선택해 가중 평균 → Anisotropic sampling
- 이미지 자리: 방향별로 다른 색상을 가진 voxel 시각화

---

## Slide 8 — ② Temporal Processing

**각 voxel에 간접광을 계산해 최종 radiance 저장**

**처리 순서 (voxel별 병렬 실행):**

```
① Voxelization 결과 언패킹
   — 13채널 atomic 데이터 → 평균화 (fragment counter로 나눔)
   — base color, emissive, direct light, normal 복원

② 이전 프레임 voxel 데이터로 Cone Tracing → 간접광 계산
   — ConeTraceDiffuse(이전 프레임 radiance, P, N)
   — 간접광 부족한 부분은 ambient로 보완

③ 최종 radiance 계산
   radiance = base_color × (direct + indirect) + emissive

④ Temporal Blending (이전 프레임 radiance와 50:50 블렌딩)
   — 깜빡임 방지, 점진적 수렴
   — 카메라 이동 시 clipmap 좌표 이동 보정

⑤ 16개 cone 방향 데이터 미리 계산 (최적화)
   — 6개 면 radiance의 가중 평균으로 미리 계산해 저장
```

- 이미지 자리: voxel radiance 시각화 (색상으로 간접광 표현)

---

## Slide 9 — ③ SDF Jump Flood

**SDF (Signed Distance Field): 각 voxel에서 가장 가까운 표면까지의 거리**

**왜 필요한가?**
- Cone Tracing 시 빈 공간에서 step size를 크게 해서 빠르게 진행
- 표면 근처에서는 step size를 줄여서 정확하게 샘플링
- → SDF 값으로 안전하게 건너뛸 수 있는 거리를 판단

**Jump Flood Algorithm:**
- 초기: 표면 voxel = 0, 빈 voxel = ∞
- 반복 (log2(해상도)번): 각 voxel에서 점점 좁아지는 간격으로 주변 탐색 → 최소 거리 업데이트
- 결과: 모든 voxel이 가장 가까운 표면까지의 정확한 거리를 가짐

```
초기:  ∞  ∞  0  ∞  ∞
결과:  2  1  0  1  2   ← 각 voxel의 표면까지 거리
```

---

## Slide 10 — Cone Tracing 기본 원리

**Cone Tracing = 특정 방향으로 점점 넓어지는 원뿔 형태로 voxel을 샘플링**

```
      ╱‾‾‾‾‾‾‾‾‾╲   ← cone 끝 (넓음, 큰 voxel 샘플링)
     ╱             ╲
    ╱               ╲
   ╱─────────────────╲  ← 중간
  ╱                   ╲
 ╱─────────────────────╲
 P ──────────────────────→  direction
  (시작점, 좁음)
```

**작동 방식:**
1. 표면 점 P에서 법선 반대 방향으로 살짝 offset (self-occlusion 방지)
2. cone 방향으로 진행하면서 voxel 샘플링
3. 거리가 멀수록 cone이 넓어짐 → 더 큰 voxel (낮은 clipmap 레벨) 샘플링
4. SDF 값으로 빈 공간에서 step size 최적화
5. Front-to-Back 블렌딩: 누적 alpha가 1에 가까워지면 종료

---

## Slide 11 — ④ Resolve Diffuse

**화면 각 픽셀에서 16개 방향으로 Cone Tracing → Diffuse GI 계산**

```
표면 법선 N 기준 반구 방향으로 16개 cone 발사
(각 cone aperture ≈ 50°, 넓은 각도)

→ 각 cone의 결과를 N과의 각도 기반 가중 합산
→ Diffuse Irradiance
```

**최적화**: Temporal Processing에서 16개 cone 방향 데이터를 미리 계산해 저장
- 직접 voxel을 3회 샘플링하는 대신 미리 계산된 cone 데이터를 1회 조회

- 이미지 자리: 16개 cone 방향 시각화 + Diffuse GI 결과

---

## Slide 12 — ⑤ Resolve Specular

**반사 방향으로 단일 narrow cone → Specular GI 계산**

```
반사 벡터 R = reflect(-V, N)  (카메라 방향의 반사)
cone aperture = roughness    (거칠수록 cone이 넓어짐)

→ R 방향으로 single cone tracing
→ Specular 반사광
```

| roughness | cone 폭 | 결과 |
|---|---|---|
| 0 (mirror) | 매우 좁음 | 선명한 반사 |
| 0.5 | 중간 | 흐릿한 반사 |
| 1 (rough) | 넓음 | diffuse에 가까운 반사 |

- DDGI가 할 수 없는 **Specular GI를 VoxelGI가 지원하는 이유**
  - 원하는 방향으로 cone을 자유롭게 발사할 수 있기 때문
- 이미지 자리: Specular reflection ON/OFF 비교

---

## Slide 13 — 전체 흐름 요약 다이어그램

```
[씬 geometry + 광원]
        ↓
[① Voxelization]
  geometry → 64³ voxel grid
  직접광 계산 후 6방향 × 13채널로 저장
        ↓
[② Temporal Processing]
  이전 프레임 voxel로 cone tracing → 간접광 계산
  final radiance = (직접 + 간접) × albedo + emissive
  temporal blending (50:50)
        ↓
[③ SDF Jump Flood]
  각 voxel → 표면까지 거리 계산 (cone tracing step size 최적화)
        ↓
[④ Resolve Diffuse]            [⑤ Resolve Specular]
  픽셀에서 16개 wide cone       반사 방향 narrow cone 1개
  → Diffuse indirect GI        → Specular indirect GI
```

---

## Slide 14 — VizMotive 구현 결과
- 이미지 자리 ×3~4장:
  - **GI OFF** vs **GI ON** 비교
  - Specular reflection 효과
  - Voxel 디버그 뷰 (radiance 시각화)
  - Color bleeding 효과

---

## Slide 15 — VoxelGI의 장점과 한계

| | 내용 |
|---|---|
| **장점** | Diffuse + **Specular GI 모두 지원** |
| **장점** | Clipmap으로 넓은 범위 커버 |
| **장점** | Ray tracing HW 불필요 (rasterization 기반) |
| **한계** | Voxel 해상도 한계 → 얇은 구조/세밀한 geometry 표현 어려움 |
| **한계** | Voxelization 비용 (매 프레임 씬을 다시 voxel로 변환) |
| **한계** | Specular는 voxel 해상도에 의해 품질 제한 |

---

## Slide 16 — 마무리 & 다음 발표 예고
- VoxelGI 핵심: **씬을 voxel로 변환 → cone tracing으로 diffuse + specular GI 계산**
- DDGI와 비교: probe 대신 voxel grid 사용, specular 추가 지원
- 다음: **SurfelGI** — 씬 표면에 surfel을 배치해 GI를 계산하는 방식
