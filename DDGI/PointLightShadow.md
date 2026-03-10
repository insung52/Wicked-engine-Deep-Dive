# Point Light Shadow 구현 수정

---

## 배경

DDGI 내부 probe 문제의 정확한 분석을 위해 point light shadow를 정상 작동시킬 필요가 있었다.

Shadow의 파이프라인 자체(shadow atlas 렌더링, cubemap camera setup, `shadow_cube()` 샘플링 셰이더 등)는 Wicked Engine에서 이미 포팅되어 있었다. 그러나 포팅 과정에서 발생한 버그 3가지로 인해 shadow가 전혀 렌더링되지 않는 상태였다.

---

## Point Light Shadow 파이프라인 구조

```
[C++]
1. CreateCubemapCameras()로 6개 face 카메라 생성
2. ShadowMap_Detail.cpp에서 각 face에 geometry 렌더링 → shadow atlas에 depth 저장
3. ShaderEntity에 shadow atlas UV 정보 + ENTITY_FLAG_LIGHT_CASTING_SHADOW 설정

[Shader]
4. lightingHF.hlsli: light.IsCastingShadow() 확인
5. shadow_cube()로 shadow atlas에서 depth 샘플링
   - cubemap_to_uv(-L)로 face 및 UV 계산
   - depth 비교 → shadow 여부 결정
```

---

## 버그 1: Shadow Atlas에 아무것도 렌더링되지 않음 (`ShadowMap_Detail.cpp`)

### 원인

6개 face 카메라를 cb(constant buffer)에 등록하기 전, 메인 카메라 frustum과 교차하는 face만 포함하는 최적화 코드가 있었다.

```cpp
// 수정 전
if (cam_frustum.Intersects(cameras[shcam].boundingfrustum))
{
    XMStoreFloat4x4(&cb.cameras[camera_count].view_projection, ...);
    camera_count++;
}
```

`cam_frustum`은 `BoundingFrustum::CreateFromMatrix()`로 생성했는데, DirectXMath의 `BoundingFrustum`은 LHS(왼손 좌표계) 기준이지만 VizMotive의 view-projection matrix는 RHS(오른손 좌표계)다. 좌표계 불일치로 `Intersects()`가 항상 false를 반환해 **camera_count가 항상 0**이 되고, shadow atlas에 아무것도 렌더링되지 않았다.

### 수정

frustum check를 제거하고 6개 face를 항상 cb에 등록한다. 메인 카메라와의 교차 여부와 관계없이 모든 face를 렌더링한다.

```cpp
// 수정 후: frustum check 없이 항상 6개 face 포함
for (uint32_t shcam = 0; shcam < arraysize(cameras); ++shcam)
{
    XMStoreFloat4x4(&cb.cameras[camera_count].view_projection, cameras[shcam].view_projection);
    cb.cameras[camera_count].output_index = shcam;
    // ...
    camera_count++;
}
```

---

## 버그 2: 큐브맵 카메라 방향 오류 (`RenderPath3D_Detail.cpp`, `shadowHF.hlsli`)

### shadow_cube()의 face 선택 방식

셰이더에서 shadow 샘플링 시 `cubemap_to_uv(-L)`로 face를 결정한다.

- `L = light.position - surface.P` (surface → light 방향)
- `-L = surface.P - light.position` (light → surface 방향)
- face 0: `-L.x > 0` → surface가 light의 **+X** 쪽에 있음

따라서 face 0의 shadow camera는 **+X 방향**을 바라봐야 shadow atlas의 해당 face에 +X 방향 depth가 저장된다.

### 원인

기존 쿼터니언은 이 규칙과 맞지 않았다.

```cpp
// 수정 전: face 방향이 cubemap_to_uv() 규칙과 일치하지 않음
shcams[0].init(position, XMFLOAT4(0.5f, -0.5f, -0.5f, -0.5f), ...); //+x
shcams[1].init(position, XMFLOAT4(0.5f,  0.5f,  0.5f, -0.5f), ...); //-x
shcams[2].init(position, XMFLOAT4(1, 0, 0, -0),                ...); //+y
shcams[3].init(position, XMFLOAT4(0, 0, 0, -1),                ...); //-y
shcams[4].init(position, XMFLOAT4(0.707f, 0, 0, -0.707f),      ...); //+z
shcams[5].init(position, XMFLOAT4(0, 0.707f, 0.707f, 0),       ...); //-z
```

### XMMatrixLookToRH의 U축 반전 문제

`XMMatrixLookToRH`는 `right = cross(up, -fwd)`로 계산하여 V(수직)는 맞지만 **U(수평)가 항상 반전**된다. shadow atlas에 기록된 U가 뒤집혀 있으므로 샘플링 시 보정이 필요하다.

### 수정

```cpp
// 수정 후: cubemap_to_uv() face 규칙에 맞는 쿼터니언
// r = -L(surface-light) 기준. 각 face는 r의 해당 방향을 바라봄.
// LookToRH: U 항상 반전 → shadowHF.hlsli에서 1-u로 보정
shcams[0].init(position, XMFLOAT4(0,-0.707f,0,0.707f), ...); // face0: +X, up=(0,1,0)
shcams[1].init(position, XMFLOAT4(0, 0.707f,0,0.707f), ...); // face1: -X, up=(0,1,0)
shcams[2].init(position, XMFLOAT4(0, 0.707f,-0.707f,0),...); // face2: +Y, up=(0,0,-1)
shcams[3].init(position, XMFLOAT4(0, 0.707f, 0.707f,0),...); // face3: -Y, up=(0,0,+1)
shcams[4].init(position, XMFLOAT4(0, 1, 0, 0),         ...); // face4: +Z, up=(0,1,0)
shcams[5].init(position, XMFLOAT4(0, 0, 0, 1),         ...); // face5: -Z, up=(0,1,0)
```

shadowHF.hlsli에서 U 반전 보정:

```hlsl
// shadow_cube() — 두 variant 모두 적용
const float3 uv_slice = cubemap_to_uv(-Lunnormalized);
float2 shadow_uv = uv_slice.xy;
shadow_uv.x = 1.0 - shadow_uv.x; // RHS LookToRH의 U 반전 보정
shadow_uv.x += uv_slice.z;       // face offset (atlas에서 해당 face 열로 이동)
```

---

## 버그 3: ShaderEntity에 CASTING_SHADOW 플래그 미설정 (`RenderPath3D_Detail.cpp`)

### 원인

셰이더에서 `light.IsCastingShadow()`로 shadow 계산 여부를 판단한다.

```hlsl
// lightingHF.hlsli
if (light.IsCastingShadow() && surface.IsReceiveShadow())
{
    shadow = shadow_cube(light, L, ...);
}
```

`IsCastingShadow()`는 ShaderEntity의 `ENTITY_FLAG_LIGHT_CASTING_SHADOW` 비트를 확인한다. 그런데 C++에서 ShaderEntity를 빌드할 때 이 플래그를 설정하는 코드가 누락되어 있었다. 결과적으로 shadow atlas에 depth가 올바르게 렌더링되더라도 셰이더에서 shadow 계산 자체를 건너뛰었다.

### 수정

```cpp
// RenderPath3D_Detail.cpp — ShaderEntity 빌드 시 추가
if (light.IsCastingShadow())
{
    shaderentity.SetFlags(ENTITY_FLAG_LIGHT_CASTING_SHADOW);
}
```
