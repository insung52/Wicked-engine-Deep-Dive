1031 분석

DirectX Raytracing 에서 LHS 에 의존하는 부분을 찾고 이를 RHS 로 수정하는 방법으로 진행

해야할 일 : DDGI raytracing 에서 LHS에 의존하는 부분 찾기

RTX 기반 ray-tracer 이해
DDGI ray-tracing 코드 리뷰

VizMotive Engine DXR 구조:

1. Scene Update (CPU)
    ↓
    SceneUpdate_Detail.cpp
    - BLAS/TLAS 생성
    - Instance flags 설정 ← 우리가 수정한 부분!
    ↓
2. Graphics Backend (CPU → GPU)
    ↓
    GraphicsDevice_DX12.cpp
    - DXR API 호출
    - D3D12 구조체 변환
    ↓
3. GPU Raytracing (GPU)
    ↓
    ddgi_raytraceCS.hlsl
    - TraceRayInline() 호출
    - Hit 결과 처리
    - CommittedTriangleFrontFace() 판정

---

# 문제 원인



### Wicked Engine

- DXR 은 기본적으로 LHS CW 를 앞면으로 인식
- wicked 엔진의 경우, LHS CCW geometry 이므로, DXR 을 사용하더라도 geometry를 CCW 그대로 인식합니다.

Wicked Engine 은 LHS CCW 를 앞면으로 geometry 를 생성하므로, 해당 코드를 사용합니다.

기존 코드 (Wicked Engine 과 Vizmotive Engine 동일):
```cpp
if (XMVectorGetX(XMMatrixDeterminant(W)) > 0)
{
  // VizMotive geometry is CCW by default, so set CCW flag for normal (non-mirrored) transforms
  instance.flags |= RaytracingAccelerationStructureDesc::TopLevel::Instance::FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
```
이 코드의 역할 : 일반적인 geometry(det>0) 에 대해, **CCW 를 앞면으로 인식하겠다.**

### Vizmotive Engine
- vizmotive engine 의 geometry 는 RHS CCW
- 일반 렌더링 시 : RHS 그대로 사용하므로 정상 작동
- DXR 은 LHS 기준으로 winding 판단 -> geometry의 앞면을 CW 로 인식! 

### 핵심 정보

**RHS CCW = LHS CW**
  
따라서, vizmotive engine 에서는 DXR (DirectX Raytracing) 사용시 CW 를 앞면으로 인식 (DXR 기본값) 하도록 CCW 플래그의 조건을 반대로 수정해야 합니다.

문제 해결 코드:
```cpp
if (XMVectorGetX(XMMatrixDeterminant(W)) < 0)
{
  // VizMotive uses RHS (Right-Handed System), but DXR operates in LHS (Left-Handed System)
  // RHS CCW geometry is interpreted as CW by DXR, matching DXR's default CW=frontface behavior
  // Only mirrored transforms (det < 0) need CCW flag: RHS CCW → mirror → RHS CW = DXR CCW
  //	https://docs.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_raytracing_instance_flags
  instance.flags |= RaytracingAccelerationStructureDesc::TopLevel::Instance::FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
}
```
이 코드의 역할 : 반사된 geometry(det<0) 에 대해, CCW 를 앞면으로 인식. 

일반적인 geometry 는 CW (기본) 를 앞면으로 인식


---
# 디버깅 과정 (기록용)

### DDGI raytracing 의 빛 방향 또는 geometry 의 표면 방향에 오류가 없음을 검증함 (문제 범위 축소) (25/10/30 디버깅)

ddgi_debugPS.hlsl 셰이더 코드를 수정하여 3개의 데이터를 rgb로 시각화

![alt text](<image copy 2.png>)

- Red 채널: isBackface (빨간색 = backface, 검은색 = frontface)

- 결과 : 메시 외부 : 빨간색, 메시 내부 : 검은색

- Green 채널: NdotL (초록색 밝기 = 조명 강도)

- 결과(외부) : 메시 근처, 조명 반대편 probe 가 초록색
- 메시 내부 : 조명이랑 가까운쪽 probe 가 초록색 (외부랑 반대)

- Blue 채널: Normal Y (파란색 밝기 = normal의 위/아래 방향)

- 결과(외부) : 바닥 메시 아래 probe 가 파란색, 위 probe 가 검은색
- 메시 내부 : 파란색 (근데 그렇게 차이는 모르겠음)

backface true -> -N 적용됨(surfaceHF.hlsli:390).

이 코드의 원래 역할 : 메시 내부의 probe 도 간접조명 정보를 얻을 수 있게하기 위함. 

결국 isBackface 가 제대로 나와야 해결됨.

해당 줄의 코드 비활성화시, Red 는 그대로, Green 은 올바른 값 나옴, Blue 도 올바른 값 나옴.

= front face 를 back face 로 인식하지만, 노말 방향을 뒤집지 않기 때문에 ddgi 가 정상적을 작동함.(????) 진짜 그 surface 가 winding 이 문제가 있어서 노말 방향이 뒤집혀 있었다면, -N 을 적용한 기존 코드에서 문제가 없었어야 되지 안나???

다시 분석

isbackface 일때 노말 방향을 뒤집었을때 ddgi 오류 (ndotL 이 반대) -> isbackface 가 true 더라도, 노말 방향을 뒤집지 않았을때 ddgi 가 정상작동을 한다 -> isBackface 랑 관계없이 해당 면의 노말은 제대로 계산되었다??? -> isBackface 가 사실 반대여야 한다???

### 경우의 수를 나누어 최종 검증

조명 -> probe -> 메시 앞면 인 경우에 대해서,

경우의 수 : N dot L 계산 시, light 의 방향이 제대로 설정되지 않은 경우 (L : light -> surface(면으로 들어가는 방향), 원래 n dot l 계산할떄는 surface -> light 로 빛 방향을 진행방향의 반대로 해야 됨.)

- 기존 : isbackface true 이므로 -N 적용 -> -N dot L 이 음수로 나옴 -> -N 은 면에서 나오는 방향이다. -> N 은 면으로 들어가는 방향이다.(기존의 반대)

- -N 코드 제거 : N dot L 이 양수 -> N 이 면으로 들어가는 방향이다(기존의 반대)

경우의 수 2 : L 방향은 문제가 없다. (N dot L 계산할때 L 방향을 실제 빛 진행방향의 반대로 잘 설정했다, surface -> light, 즉 면에서 나오는 방향)

- 기존 : isbackface true 이므로 -N 적용 -> -N dot L 이 음수 -> -N 이 면으로 들어가는 방향 -> N 은 면에서 나오는 방향.(정상)

- -N 코드 제거 : N dot L 이 양수 -> N 이 면에서 나오는 방향이다.(정상)

**결론 : 가능한 경우의 수 : (빛의 방향이 정상 and N 방향도 정상) or (빛의 방향이 비정상 and N 방향도 비정상)**

**따라서 빛의 방향과 N 방향 에 따른 처리 코드는 문제가 없다!**

더 조사해야될 부분 : isBackface 는 대체 누구 기준으로 계산하는지? probe 가 어떠한 삼각형을 보았을때 그 삼각형이 backface 인지 frontface 인지 확인하나? 그 코드에 문제가 있는가?

---

backface 결정 코드 : surface.SetBackface(!q.CommittedTriangleFrontFace()); (ddgi_raytraceCS.hlsl:265)

이 코드에서 ! 를 제거하면 놀랍게도 ddgi 가 정상적으로 작동함.

허나 그 현상에 대한 합리적이고 정확하고 본질적인 이유를 알아내야함.

q : probe 에서 출발하는 아무 방향의 raytracing 결과?
```cpp
		RayDesc ray;
		ray.Origin = probePos;
		ray.TMin = 0; // don't need TMin because we are not tracing from a surface
		ray.TMax = FLT_MAX;
		ray.Direction = normalize(raydir);

#ifdef RTAPI
		vzRayQuery q;
		q.TraceRayInline(
			scene_acceleration_structure,	// RaytracingAccelerationStructure AccelerationStructure
			//RAY_FLAG_CULL_BACK_FACING_TRIANGLES |
			RAY_FLAG_SKIP_PROCEDURAL_PRIMITIVES |
			RAY_FLAG_FORCE_OPAQUE,			// uint RayFlags
			push.instanceInclusionMask,		// uint InstanceInclusionMask
			ray								// RayDesc Ray
		);
		while (q.Proceed());
		if (q.CommittedStatus() != COMMITTED_TRIANGLE_HIT)
```
이 q 에서 사용하는 코드들에 문제가 있을 수 있다. ex. scene_acceleration_structure, ray
이 부분에 대해서 제대로 확인. ex (레이트레이싱 가속 구조 생성시 rhs, lhs 별로 방법이 다르다거나?)

---

1. light direction 계산 - ndotL 계산할때, L 의 방향이 표면에서 light source 로 가는 방향이어야 함. 즉 lgith 의 실제 입사 방향과 반대여야함. (DDGI 의 경우 surface -> probe 방향으로 바꿔야함) 이 코드가 구현이 되어있는가?

  1. CPU Side (BasicComponents.cpp)
```cpp
  const XMVECTOR BASE_LIGHT_DIR = XMVectorSet(0, 0, -1, 0); // Forward direction
  // 빛이 나아가는 방향 (light → surface)
  // 예: 태양이 위에서 아래로 빛을 쏨 → (0, 0, -1)

  XMStoreFloat3(&direction, XMVector3Normalize(XMVector3TransformNormal(BASE_LIGHT_DIR, W)));
  // light.direction = 빛의 진행 방향
```
  2. GPU Upload (RenderPath3D_Detail.cpp:920-921)
```cpp
  // note: the light direction used in shader refers to the direction to the light source
  shaderentity.SetDirection(XMFLOAT3(-light.direction.x, -light.direction.y, -light.direction.z));
```
  여기서 부호 반전
  - CPU의 light.direction: (0, 0, -1) (빛이 아래로 내려감)
  - GPU로 전달: (0, 0, +1) (표면에서 빛을 향해 위로 올라감)

  3. Shader (ddgi_raytraceCS.hlsl, lightingHF.hlsli)
```cpp
  L = light.GetDirection().xyz;  // (0, 0, +1) = surface → light
  NdotL = saturate(dot(L, surface.N));
```

2. GeometryGenerator.cpp 에서 CW winding 하는 코드를 CWW 로 변경해야함. vizmotive 는 CWW winding 으로 하기로 결정했음.

적용후, ddgi probe 들은 제대로 조명 계산을 하지만, 카메라에서 뒷면만 보이는 새로운 문제가 발생함!

![alt text](<image copy.png>)


3. mesh importer 에서 읽어들인 데이터를 CPU, GPU 에 저장할때, winding convention 을 바꾸는 경우가 있는지 확인해야함.


---

이전 문제 분석 내용 정리

### 1. DDGI 문제 분석

바닥 메시 있을때 - 바닥 메시 아래의 probe 가, 바닥 메시의 아랫면을 무시하고, 바닥 메시 윗면의 밝은 간접광을 샘플링해서 밝아짐

![alt text](i0.png)

바닥 메시 비활성화 시 - 다시 어두워짐

![alt text](i1.png)

Wicked Engine 에서, 문제없음.

![alt text](i2.png)

거리 텍스쳐 적용 (가까움 : 파란색, 멈 : 빨간색), 거리 텍스쳐에는 문제가 없는 듯 하다. (바닥 메시를 잘 인식함)

![alt text](i3.png)

바닥 메시 윗면의 반사광을 샘플링 해야할 위쪽 probe 들이, 오히려 어두운 현상 발견

![alt text](i4.png)

빛을 받는 메시의 면 근처보다, 그 반대 면 근처 probe 들이 더 밝음. 둘이 바뀐거 같음 -> probe 에서 출발한 ray 가, 메시에서 바깥으로 나가는 방향에서만 radiance 를 샘플링 한다는 1차 결론

![alt text](i5.png)

probe 들이 메시의 앞면에 맞은 경우 (if (surface.IsBackface()==True)  ) 파란색, 뒷면에 맞은 경우 빨간색으로 디버깅한 결과

![alt text](i6.png)

결론 : 모든 면들의 노멀 방향이 반대다! Surface 의 is_backface 를 결정하는 로직을 변경해야한다. (좌표계에 맞게)

---

# 2. DDGI 문제 해결

## 2.1. ROOT CAUSE 분석

### 문제의 핵심
VizMotive와 Wicked Engine의 **geometry winding order 차이**로 인한 DXR raytracing 결과 불일치

### Geometry Winding Order란?
- **CW (Clockwise)**: 삼각형 vertex를 시계방향으로 정의
- **CCW (Counter-Clockwise)**: 삼각형 vertex를 반시계방향으로 정의
- 외부에서 볼 때 어느 방향이 "앞면"인지 결정

### DXR의 기본 규칙
- **기본 (플래그 없음)**: CW winding = frontface
- **`D3D12_RAYTRACING_INSTANCE_FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE` 설정 시** (반사된 지오메트리?):
  CCW winding = frontface

### VizMotive vs Wicked Engine

| 엔진 | Geometry Winding | CCW Flag 설정 조건 | 결과 |
|------|------------------|-------------------|------|
| **Wicked Engine** | CCW | `det > 0` | ✓ 정상: CCW 지오메트리 + CCW 플래그 |
| **VizMotive (원래)** | CW | `det > 0` | ✗ 오류: CW 지오메트리 + CCW 플래그 |
| **VizMotive (수정 후)** | CW | `det < 0` | ✓ 정상: CW 지오메트리, 플래그 없음
  |

**Wicked Engine 은 CCW 지오메트리를 기본값으로 사용 -> 지오메트리에 CCW Flag 를 적용 -> CCW 가 앞면이 되도록 설정.**

**반대로 VizMotive 는 CW 지오메트리를 사용하고, 동시에 Wicked Engine 의 코드를 사용했기 때문에 오류가 발생!** 

### 코드 분석

**VizMotive의 Box Geometry 생성** (`GeometryGenerator.cpp:663-669`):
```cpp
// 첫 번째 삼각형: a-b-d
indices.push_back(a);  // 좌하단 (-0.5, -0.5, +0.5)
indices.push_back(b);  // 좌상단 (-0.5, +0.5, +0.5)
indices.push_back(d);  // 우하단 (+0.5, -0.5, +0.5)

카메라에서 볼 때:
    b(좌상)
    ↑ ↘
    a(좌하) → d(우하)
→ CW (Clockwise) winding 확인
```

원래 플래그 설정 로직 (SceneUpdate_Detail.cpp:707-713):
```cpp
float det = XMVectorGetX(XMMatrixDeterminant(W));
if (det > 0)  // ← 문제: 정상 transform에서 CCW 플래그 설정
{
    instance.flags |= FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
}
```

결과:
- VizMotive의 모든 객체: det > 0 → CCW 플래그 SET
- DXR 해석: CCW = frontface
- 실제 지오메트리: CW
- DXR이 모든 frontface를 backface로 잘못 인식!

Shader에서의 영향

RTAPI 경로 (ddgi_raytraceCS.hlsl:262):
```
surface.SetBackface(!q.CommittedTriangleFrontFace());
```

원래 상황 (CCW 플래그가 잘못 설정됨):
1. 실제 앞면 (CW) hit
2. DXR: CCW 플래그 때문에 CW를 backface로 해석
3. CommittedTriangleFrontFace() returns false
4. !false → SetBackface(true) → 앞면이 backface로 잘못 설정
5. surfaceHF.hlsli:390-393에서 노멀 뒤집힘
6. 조명 계산 오류 → probe가 반대 방향 밝기를 샘플링

## 2.2. 해결 방법

TLAS Instance Flag 조건 수정

파일: EngineShaders/ShaderEngine/SceneUpdate_Detail.cpp

변경 전: (일반 지오메트리에 CCW flag 적용)
```cpp
float det = XMVectorGetX(XMMatrixDeterminant(W));
if (det > 0)
{
    // There is a mismatch between object space winding and BLAS winding:
    instance.flags |= RaytracingAccelerationStructureDesc::TopLevel::Instance::
FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
}
```

변경 후: (반사 지오메트리에만 CCW flag 적용)
```cpp
float det = XMVectorGetX(XMMatrixDeterminant(W));
if (det < 0)
{
    // Mirrored/reflected transform: flip winding interpretation
    // VizMotive geometry is CW by default, so only set CCW flag when mirrored
    instance.flags |= RaytracingAccelerationStructureDesc::TopLevel::Instance::
FLAG_TRIANGLE_FRONT_COUNTERCLOCKWISE;
}
```
설명:
- det < 0: Transform이 mirror/reflection을 포함 → winding이 반대로 변환됨
- 이 경우에만 CCW 플래그를 설정하여 winding을 다시 원래대로 복원
- 정상 transform (det > 0)에서는 플래그 없음 → DXR이 CW를 frontface로 올바르게 해석

## 2.3. 검증

Transform Determinant 이해

Determinant > 0:
- Uniform scale, rotation만 포함
- Winding order 유지
- VizMotive 기본 상태

Determinant < 0:
- Mirror, reflection 포함
- Winding order 반전 (CW ↔ CCW)
- 이 경우에만 CCW 플래그 필요

수정 후 동작

| Transform | det | CCW Flag | Geometry | DXR 해석      | 결과  |
|-----------|-----|----------|----------|-------------|-----|
| Normal    | > 0 | 미설정      | CW       | CW = front  | ✓   |
| Mirrored  | < 0 | 설정       | CW → CCW | CCW = front | ✓   |