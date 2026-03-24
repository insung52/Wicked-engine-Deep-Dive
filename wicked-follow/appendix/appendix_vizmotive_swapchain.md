# Appendix: VizMotive SwapChain 구조 분석

> 이 문서를 읽기 전 권장 선행 자료:
> - [part3_gpu_communication.md § 3.8](../../study/graphics/part3_gpu_communication.md) — 스왑체인 기초, DXGI_USAGE 플래그, 중간 RT 패턴
> - [appendix_swapchain_srv_comparison.md](appendix_swapchain_srv_comparison.md) — WickedEngine 변경 전/후 비교

---

## VizMotive 현재 SwapChain_DX12 구조

```cpp
// GraphicsDevice_DX12.cpp:1494
struct SwapChain_DX12
{
    std::shared_ptr<AllocationHandler>       allocationhandler;
    ComPtr<IDXGISwapChain3>                  swapChain;        // DXGI 스왑체인 객체
    std::vector<ComPtr<ID3D12Resource>>      backBuffers;      // 버퍼당 Raw resource
    std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferRTV;    // 버퍼당 RTV
    Texture dummyTexture;    // GetBackBuffer() 반환용 임시 객체
    ColorSpace colorSpace = ColorSpace::SRGB;

    inline uint32_t GetBufferIndex() const {
        return swapChain->GetCurrentBackBufferIndex();
    }
};
```

**현재 상태 = WickedEngine 변경 전 구조와 동일.**
- SRV 없음 → 셰이더에서 backbuffer 읽기 불가
- `DXGI_USAGE_RENDER_TARGET_OUTPUT`만 설정 (GraphicsDevice_DX12.cpp:3162)

---

## GetBackBuffer() 현재 동작

```cpp
// GraphicsDevice_DX12.cpp:5930
Texture GraphicsDevice_DX12::GetBackBuffer(const SwapChain* swapchain) const
{
    auto swapchain_internal = to_internal(swapchain);

    // ⚠️ 매 호출마다 Texture_DX12 힙 할당
    auto internal_state = std::make_shared<Texture_DX12>();
    internal_state->allocationhandler = allocationhandler;
    internal_state->resource = swapchain_internal->backBuffers[currentIndex];

    // footprints 계산 (READBACK 복사용 — 스크린샷 경로에서 필요)
    device->GetCopyableFootprints(&resourcedesc, 0, mipCount, 0,
        footprints.data(), numRows.data(), rowSizesInBytes.data(), &total_size);

    Texture result;
    result.type = GPUResource::Type::TEXTURE;
    result.internal_state = internal_state;  // 새 객체
    result.desc = _ConvertTextureDesc_Inv(resourcedesc);
    return result;   // SRV 없음
}
```

`GetBackBuffer()`는 현재 `Helpers2.cpp:64`에서만 쓰인다 — `saveTextureToFile()` (스크린샷 저장).

---

## VizMotive 포스트프로세싱 흐름

```
┌─────────────────────────────────────────────────────────────┐
│                     RenderProcess()                         │
│                                                             │
│  [Scene Render]                                             │
│       ↓                                                     │
│  rtMain (R11G11B10_FLOAT, HDR)  ← 씬 렌더링 결과              │
│       ↓                                                     │
│  [TAA Compute]  (선택, ping-pong)                            │
│       ↓                                                     │
│  [Postprocess_Tonemap]          ← Compute Shader            │
│    input:  rtMain SRV           (HDR float 읽기)             │
│    output: rtPostprocess UAV    (LDR UNORM 쓰기)             │
│       ↓                                                     │
│  lastPostprocessRT = rtPostprocess                          │
│       ↓                                                     │
│  ┌─── Compose() ────────────────────────────────────────┐   │
│  │ RenderPassBegin(&swapChain_, cmd)  ← 스왑체인 RTV     │   │
│  │   image::Draw(lastPostprocessRT)   ← Graphics Blit   │   │
│  │     [Pixel shader: lastPostprocessRT SRV 읽기         │   │
│  │      → swapchain RTV에 픽셀 출력]                      │   │
│  │ RenderPassEnd()                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│       ↓                                                     │
│  [HDR10 경로만 추가 패스]                                     │
│  RenderPassBegin(&swapChain_, cmd)                          │
│    image::Draw(&rtRenderFinal_, HDR10 fx)                   │
│  RenderPassEnd()                                            │
│       ↓                                                     │
│  Present()                                                  │
└─────────────────────────────────────────────────────────────┘
```

**핵심 관계:**
- `rtMain` → **Compute** tonemap → `rtPostprocess` (중간 RT, LDR)
- `rtPostprocess` → **Graphics blit** → `swapchain` (최종 출력)
- `swapchain`은 항상 **쓰기 대상(RTV)** — 읽는 코드 없음

---

## 왜 중간 RT(`rtPostprocess`)가 필요한가

| 이유 | 설명 |
|------|------|
| **포맷 불일치** | `rtMain`: R11G11B10_FLOAT (HDR), `swapchain`: R8G8B8A8_UNORM (LDR) |
| **Compute → UAV 필요** | Tonemapper는 compute shader → output이 UAV여야 함. 스왑체인은 UAV 불가. |
| **Graphics blit 필요** | 최종 swapchain 출력은 graphics pipeline(`image::Draw`). 중간 RT가 SRV input이 됨. |

```
rtMain (HDR float) → [compute, UAV] → rtPostprocess (LDR UNORM)
                                           ↓ SRV
                     [graphics blit] ← (여기서 읽음)
                           ↓ RTV
                       swapchain
```

---

## 구조 최적화 시 무엇이 달라지는가

### WickedEngine vs VizMotive 접근 비교

WickedEngine은 구조 통합 시 SRV도 함께 추가했다.
코드를 직접 확인한 결과, **WickedEngine에서도 backbuffer SRV를 셰이더에서 직접 읽는 곳은 없다**.
(CrossFade와 스크린샷은 모두 `CopyResource` 방식 사용 — SRV 불필요)

VizMotive는 불필요한 SRV를 생략하고 구조 통합만 적용한다.

| 항목 | 현재 VizMotive | WickedEngine 방식 | VizMotive 적용 방식 |
|------|--------------|-----------------|-------------------|
| 내부 구조 | `backBuffers[]` + `backbufferRTV[]` + `dummyTexture` | `textures[]` (RTV + SRV) | `textures[]` (RTV만) |
| `GetBackBuffer()` 비용 | 매 호출 `make_shared` 힙 할당 | 참조 반환 (할당 0) | 참조 반환 (할당 0) |
| DXGI_USAGE 플래그 | `RENDER_TARGET_OUTPUT`만 | `+ SHADER_INPUT` 추가 | 변경 없음 |
| SRV | 없음 | 있음 (미사용) | 없음 (의도적 생략) |

### 달라지지 않는 것 (현재 렌더링 파이프라인)

| 항목 | 이유 |
|------|------|
| Tonemap → `rtPostprocess` 중간 RT | Compute + UAV 방식이라 swapchain 직접 쓰기 불가 |
| 최종 `image::Draw` blit | Graphics pipeline이라 그대로 유지 |
| HDR10 `rtRenderFinal_` 중간 RT | compose 결과 저장 RT 여전히 필요 |

**결론**: 현재 VizMotive 렌더링 파이프라인은 backbuffer를 셰이더에서 읽지 않는다.
구조 통합으로 얻는 이점:
1. **`GetBackBuffer()` 힙 할당 제거** — 참조 반환으로 변경
2. **코드 구조 정리** — 분산된 3개 컨테이너 → `textures[]` 통합
3. **SRV 생략** — `DXGI_USAGE_SHADER_INPUT` 없음, descriptor 슬롯 절약

---

## 적용 판단

VizMotive에 적용할 내용: **`textures[]` 구조 통합 + `GetBackBuffer()` 참조 반환, SRV 없음**

→ 구체적인 코드 변경: `apply_texture.md 변경 3` 참고
