# DDGI Raytracing 문제 분석

## 현재 문제 상황

두 지오메트리가 약간 겹쳐 있을 때, 빛이 닿을 수 없는 위치에 간접광이 보임.

![alt text](image-2.png)

왼쪽 구 2개 : 구의 그림자가 바닥 메시에 없음 + DDGI 간접광 = 빛을 직접 받는 바닥 메시보다 더 밝아지는 결과
- point light 에도 shadow 를 추가하면 자연스러운 해결 가능.

오른쪽 구 1개 : 빛이 생기면 안되는 위치 (구, 바닥 메시로 가려지는 부분)에 간접광 발생
- point light shadow 와 관련이 낮은 문제 발견


### 추가 분석

<https://github.com/user-attachments/assets/1858a026-f6c8-4f0a-bead-4a481cd0f258>

구 geometry 내부의 probe 가 간접광을 받고있는 상황을 발견
- geometry 내부 probe 들 중, 빛과 가까운 쪽 probe 보다 반대쪽 probe 들이 더 밝음!

![alt text](image-3.png)

wicked engine 의 경우 지오메트리 내부 probe 검은색

---

## 확인된 동작

### Point Light 직접광

- **shadow 미구현** (현재)
- **Back face는 NdotL = 0으로 처리** → 직접광 자체는 back face를 조명하지 않음
- 즉, point light 직접광이 geometry 내부면을 직접 밝히는 일은 없음

### DDGI Ray 충돌 대상

- `RAY_FLAG_CULL_BACK_FACING_TRIANGLES` 주석 처리됨 → **front/back face 모두 hit 가능**
- Closest hit만 사용 (가장 가까운 면 하나만)
- Back face hit 시 normal을 반전: `N = -N` (`surfaceHF.hlsli:390-393`)
- `surface.facenormal`은 이 반전 이후에 설정됨 → `facenormal == surface.N` (back face에서도 동일)

### Back face hit 시 depth 처리

```hlsl
if (surface.IsBackface()) {
    hit_depth *= 0.9;  // 내부로 약간 당김
}
```

---

## 근본 원인: Back Face Normal Flip이 만드는 역전 현상

### 예시 설정

```
구 geometry (반지름 R, 중심 원점)
point light: +X 방향 외부
probe A: 구 내부 그늘쪽 (-X 방향)
probe B: 구 내부 밝은쪽 (+X 방향)
```

### Probe A (그늘쪽 내부)의 ray 업데이트 과정

```
1. Probe A에서 -X 방향으로 ray 발사
2. 구 그늘쪽 내벽 hit (back face)
3. 기하 outward normal: (-1, 0, 0)
4. back face flip → surface.N = (+1, 0, 0)  ← 광원 방향을 가리킴
5. ddgi_sample_irradiance(hit_pos, facenormal=(+1,0,0), ...)
   └─ SH::CalculateIrradiance(sum_sh, N=(+1,0,0))
   └─ +X hemisphere의 irradiance 추출 → 밝은 외부 probe들의 SH → 밝음
6. 이 radiance가 Probe A의 ray buffer에 저장 → Probe A가 밝아짐
```

### Probe B (밝은쪽 내부)의 ray 업데이트 과정

```
1. Probe B에서 +X 방향으로 ray 발사
2. 구 밝은쪽 내벽 hit (back face)
3. 기하 outward normal: (+1, 0, 0)
4. back face flip → surface.N = (-1, 0, 0)  ← 그늘 방향을 가리킴
5. ddgi_sample_irradiance(hit_pos, facenormal=(-1,0,0), ...)
   └─ SH::CalculateIrradiance(sum_sh, N=(-1,0,0))
   └─ -X hemisphere의 irradiance 추출 → 어두운 외부 probe들의 SH → 어두움
6. Probe B가 상대적으로 어두워짐
```

### 결과

**그늘쪽 내부 probe가 밝은쪽 내부 probe보다 더 밝아지는 역전 현상 발생.**
Back face normal flip이 SH 평가 방향을 반전시켜, 물리적으로 틀린 irradiance를 샘플링하기 때문.

> `ddgi_sample_irradiance`는 최종 화면 렌더링(`shadingHF.hlsli`)에서도 쓰이지만,
> 여기서의 원인은 probe ray 업데이트(`ddgi_raytraceCS.hlsl:426`) 중 hit surface에서 호출되는 것이다.
> Hit surface는 screen pixel이 아니라 probe ray가 맞은 내부벽 지점.

---

## Probe Relocation의 한계

`ddgi_updateCS.hlsl`의 relocation 로직:

```hlsl
// 각 ray가 너무 가까이 hit → 반대 방향으로 probe를 밀기
if (depth < probeOffsetDistance)
{
    probeOffsetNew -= ray.direction * (probeOffsetDistance - depth);
}
// 최대 이동량: ddgi_cellsize() * 0.5 (그리드 셀 절반)
probeOffset = clamp(probeOffset, -probe_limit, probe_limit);
```

**닫힌 geometry 내부에 완전히 갇힌 probe의 경우:**
- 모든 방향의 ray가 내벽에 hit → 각 ray의 반발력이 서로 상쇄 → 합력 ≈ 0
- 최대 이동량 제한으로 인해 탈출 불가
- voxelgrid fallback이 있으나 voxelgrid가 활성화된 경우에만 동작

**결론: Relocation은 벽 근처 probe를 표면에서 띄우는 용도이며, 닫힌 solid geometry 내부에 갇힌 probe를 탈출시키지 못한다.**

---

## 해결 방향

### 1. Point Light Shadow 구현 (근본 해결)
- 외부면에서 빛이 차단되면 내부로 직접광이 들어올 수 없음
- Probe ray의 shadow ray 판정이 올바르게 작동하면 내부면에서 잘못된 radiance 축적 방지
- 겹친 geometry에서 간접광 누수 문제도 어느정도 해결됨

### 2. Solid Geometry 내부 Probe Disable ★ 업계 표준 접근법

**DDGI 논문(Majercik et al., 2019)과 주요 게임 엔진(UE5 Lumen, Unity HDRP 등)에서 공통적으로 채택하는 방법이다.**

Solid geometry 내부에 들어간 probe는 신뢰할 수 없는 irradiance를 축적하므로 비활성화하고, 샘플링 시 유효한 주변 probe들의 irradiance만 사용한다.

**판정 기준:**
- Ray hit distance가 모든 방향에서 일정 기준 이하 (사방이 막힘) → 내부 probe로 판정
- 또는 voxelgrid로 probe 위치가 solid voxel 내부임을 직접 확인

**비활성화 probe 처리:**
- 해당 probe의 weight를 0으로 설정 → `ddgi_sample_irradiance`의 trilinear 보간에서 자동으로 제외
- 주변 유효 probe들의 irradiance로 대체

현재 구현에서는 voxelgrid fallback이 일부 역할을 하고 있으나, voxelgrid 비활성화 시 동작하지 않아 불완전하다.


ddgi 부분에서 교수님꼐서 발견하신 문제에 추가로 간접광이 비정상적으로 작동하는 부분에 대해서 조사한 결과입니다. 해결 방법으로 point light 의 shadow 를 작동하게 구현 및 solid geometry 내부 probe 를 disable 하는 로직이 있습니다. 