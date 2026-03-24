# Appendix: Render Target 압축 (RT Compression)

> 이 문서를 읽기 전 권장 선행 자료:
> - [part3_gpu_communication.md § 3.8](../../study/graphics/part3_gpu_communication.md) — DXGI_USAGE 플래그, RTV vs SRV
> - [part4_resource_management.md](../../study/graphics/part4_resource_management.md) — GPU 리소스 관리 기초

---

## GPU가 RT 데이터를 압축하는 방식

GPU는 Render Target에 픽셀을 쓸 때 raw 값을 VRAM에 그대로 저장하지 않는다.
드라이버와 하드웨어가 **자동으로** 압축을 적용한다. 코드에서 별도로 요청하지 않아도 된다.

```
GPU 픽셀 쓰기 → [하드웨어 압축] → VRAM (압축된 형태)
                                         ↓
                               디스플레이 하드웨어가 직접 읽음 (압축 해제 없이)
```

이 압축은 메모리 대역폭을 줄이고 캐시 효율을 높이는 핵심 최적화다.

---

## 압축 종류

### 공간 압축 (Spatial Compression)

인접한 픽셀 블록(4×4, 8×8)의 유사성을 활용해 압축.

```
하늘 영역 (비슷한 파란색):
[0x4A7FFF][0x4A80FF][0x4A7EFF][0x4A80FF]
[0x4A7FFF][0x4A81FF][0x4A7FFF][0x4A80FF]
→ "대부분 0x4A80FF, 약간 변동" 형태로 압축 가능
```

균일한 색상 영역(하늘, 바닥, 벽면)에서 효과적.
swapchain 버퍼에도 적용 가능 — UI, 배경이 넓게 있을 경우 유용.

### 델타 압축 (Delta Compression)

이전 프레임과의 차이만 저장하는 방식.

```
Frame N:   [픽셀값들]
Frame N+1: [Frame N과의 차이값들]  ← 움직이는 물체 근처만 변화
```

카메라가 고정된 상황에서 매우 효과적. 카메라나 씬이 크게 움직이면 효율 급감.

---

## Swapchain 버퍼에서 압축 효율이 낮은 이유

```
일반 RT (GBuffer 등):
  프레임 내에서 여러 번 쓰고 읽고 재사용
  → 압축/해제 사이클이 많이 발생
  → 압축으로 절약되는 대역폭이 큼

Swapchain 버퍼:
  프레임당 1회 풀스크린 blit (최종 출력)
  이전 프레임 데이터와 완전히 다른 새 화면
  → 델타 압축 효율 낮음
  → 공간 압축만 적용 가능 (영역에 따라 다름)
```

따라서 swapchain 버퍼의 압축 이득은 일반 RT에 비해 제한적이다.

---

## DXGI_USAGE_SHADER_INPUT이 압축에 미치는 영향

셰이더 유닛(compute, pixel shader)은 **압축된 RT 데이터를 직접 읽을 수 없다**.
하드웨어가 사용하는 압축 포맷이 셰이더 접근과 호환되지 않기 때문이다.

`DXGI_USAGE_SHADER_INPUT` 플래그를 설정하면 드라이버가 두 가지 중 하나를 선택:

| 옵션 | 동작 | 비용 |
|------|------|------|
| **압축 비활성화** | 버퍼를 항상 raw 형태로 유지 | 압축 이득 없음 |
| **온디맨드 압축 해제** | barrier 전환 시 GPU가 해제 패스 실행 | 전환마다 추가 GPU 작업 |

드라이버가 어떤 옵션을 선택하는지는 GPU 아키텍처와 드라이버 버전에 따라 다르다.

```
RENDER_TARGET_OUTPUT만:
  쓰기 → [자동 압축] → VRAM → Present (디스플레이 직접 읽기)
  빠른 경로, 추가 비용 없음

RENDER_TARGET_OUTPUT | SHADER_INPUT:
  쓰기 → [압축 제한 또는 해제 패스 필요] → VRAM → 셰이더 가능
  약간의 추가 비용 발생
```

---

## 실제 성능 영향

### Swapchain 버퍼 기준

swapchain 버퍼는 프레임당 1회 쓰이고 즉시 Present된다.
압축 이득 자체가 크지 않으므로 `SHADER_INPUT` 추가로 인한 손실도 작다.

```
압축 이득:    작음 (1회 쓰기, 델타 효율 낮음)
SHADER_INPUT 손실: 작음 (압축 이득이 작으니 잃을 것도 적음)
→ 전체 성능 영향: 미미
```

이것이 WickedEngine이 swapchain에 `DXGI_USAGE_SHADER_INPUT`을 추가하면서도
성능을 크게 신경 쓰지 않은 이유다.

### 일반 GBuffer RT 기준 (비교용)

GBuffer(노멀, 알베도, 깊이 등)는 프레임 내에서 여러 패스에서 읽히고 쓰인다.
압축 이득이 크므로 `SHADER_INPUT` 여부가 성능에 더 민감하게 작용한다.

---

## 요약

| 항목 | 내용 |
|------|------|
| RT 압축 | 드라이버/하드웨어가 자동으로 적용, 코드 불필요 |
| 공간 압축 | 인접 픽셀 유사성 활용, swapchain에도 적용 |
| 델타 압축 | 프레임 간 차이 활용, swapchain에서 효율 낮음 |
| `SHADER_INPUT` 비용 | 압축 제한/해제 패스 → 성능 영향 있지만 swapchain은 미미 |
| VizMotive 결정 | swapchain에 `SHADER_INPUT` 생략 → 미미하지만 불필요한 비용 제거 |
