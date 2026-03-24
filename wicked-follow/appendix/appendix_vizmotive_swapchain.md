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

## SRV 지원 추가 시 무엇이 달라지는가

### 달라지는 것 (구조적 개선)

| 항목 | 현재 | SRV 추가 후 |
|------|------|-------------|
| `SwapChain_DX12` 내부 | `backBuffers[]` + `backbufferRTV[]` + `dummyTexture` | `textures[]` 통합 |
| `GetBackBuffer()` 비용 | 매 호출 `make_shared` 힙 할당 | 기존 `textures[]` 참조 반환 |
| DXGI_USAGE 플래그 | `RENDER_TARGET_OUTPUT`만 | `+ SHADER_INPUT` 추가 |
| 스크린샷 효율 | READBACK 복사 필요 | 잠재적으로 SRV 직접 활용 가능 |

### 달라지지 않는 것 (현재 렌더링 파이프라인)

| 항목 | 이유 |
|------|------|
| Tonemap → `rtPostprocess` 중간 RT | Compute + UAV 방식이라 swapchain 직접 쓰기 불가 |
| 최종 `image::Draw` blit | Graphics pipeline이라 그대로 유지 |
| HDR10 `rtRenderFinal_` 중간 RT | compose 결과 저장 RT 여전히 필요 |

**결론**: 현재 VizMotive 렌더링 파이프라인은 backbuffer를 셰이더에서 읽는 코드가 없다.
SRV 지원을 추가해도 **현재 렌더링 로직 자체는 바뀌지 않는다.**

다만 다음 경우에 즉시 활용 가능:
1. **스크린샷 효율 개선** — `GetBackBuffer()` 힙 할당 제거
2. **코드 구조 정리** — `backBuffers/backbufferRTV/dummyTexture` → `textures[]` 통합
3. **미래 효과 기반** — 화면 읽기 기반 굴절/반사/오버레이 효과 추가 시 인프라 준비

---

## 적용 판단 체크리스트

```
□ 렌더링 파이프라인을 backbuffer 읽기 방향으로 바꿀 계획이 있는가?
    YES → SRV 지원 추가 필수
    NO  → 구조 정리 목적으로만 적용 여부 결정

□ 스크린샷 빈도가 높은가? (매 프레임 또는 자주)
    YES → GetBackBuffer() 힙 할당 제거 효과 있음
    NO  → 영향 미미

□ WickedEngine 코드와 최대한 일치시키고 싶은가?
    YES → 적용 권장 (구조 동기화)
    NO  → 필요할 때 적용
```

→ `apply_texture.md 변경 3` 참고
