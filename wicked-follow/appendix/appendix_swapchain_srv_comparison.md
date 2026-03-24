# Appendix: WickedEngine SwapChain — SRV 지원 전/후 구조 비교

> 이 문서를 읽기 전 권장 선행 자료:
> - [part3_gpu_communication.md § 3.8](../../study/graphics/part3_gpu_communication.md) — 스왑체인 기초, DXGI_USAGE 플래그, RTV vs SRV
> - [apply_texture.md 변경 3](../topics/apply_texture.md)

---

## 변경 전 구조

### SwapChain_DX12 (변경 전)

```cpp
struct SwapChain_DX12
{
    ComPtr<IDXGISwapChain3>              swapChain;
    std::vector<ComPtr<ID3D12Resource>>  backBuffers;     // Raw resource만
    std::vector<D3D12_CPU_DESCRIPTOR_HANDLE> backbufferRTV; // RTV만 존재
    Texture dummyTexture;                                 // GetBackBuffer용 더미
    ColorSpace colorSpace = ColorSpace::SRGB;
};
```

**문제점:**
- `backbufferRTV[]` : RTV(쓰기)만 있고 SRV(읽기) 없음
- `dummyTexture` : `GetBackBuffer()` 반환을 위한 임시 Texture 객체 보관용. 실제 의미 있는 상태가 없음
- SRV가 없으므로 bindless descriptor index가 없음 → 셰이더에서 backbuffer 직접 참조 불가

### CreateSwapChain (변경 전)

```cpp
DXGI_SWAP_CHAIN_DESC1 swapChainDesc = {};
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;  // RTV만 선언
// DXGI_USAGE_SHADER_INPUT 없음 → SRV 생성 불가
```

버퍼 초기화:
```cpp
// 버퍼별로 GetBuffer + CreateRenderTargetView만
swapChain->GetBuffer(i, PPV_ARGS(backBuffers[i]));
device->CreateRenderTargetView(backBuffers[i].Get(), &rtvDesc, backbufferRTV[i]);
// SRV는 생성 안 함
```

### GetBackBuffer() (변경 전)

```cpp
Texture GetBackBuffer(const SwapChain* swapchain)
{
    auto internal = to_internal(swapchain);

    // 호출할 때마다 새로운 Texture_DX12 힙 할당
    auto state = std::make_shared<Texture_DX12>();
    state->resource = internal->backBuffers[currentIndex];

    // footprints만 채움 (복사/readback용)
    device->GetCopyableFootprints(&resourcedesc, ...);

    // SRV index 없음 → bindless에서 참조 불가
    Texture result;
    result.internal_state = state;  // 매번 새 객체
    return result;
}
```

**매 호출마다 `make_shared<Texture_DX12>()` 힙 할당 발생.**
스크린샷이나 UI 시스템에서 매 프레임 호출 시 불필요한 할당이 반복된다.

---

## 변경 후 구조

### SwapChain_DX12 (변경 후)

```cpp
struct SwapChain_DX12
{
    ComPtr<IDXGISwapChain3> swapChain;
    std::vector<Texture>    textures;    // 버퍼당 완전한 Texture 객체 (resource + RTV + SRV)
    ColorSpace colorSpace = ColorSpace::SRGB;
    // backBuffers, backbufferRTV, dummyTexture 모두 제거
};
```

각 `textures[i]`는 초기화 시 완전히 구성된 `Texture_DX12` 내부 상태를 가진다:
```
textures[i].internal_state (Texture_DX12):
    ├─ resource   → ID3D12Resource (백버퍼)
    ├─ rtv        → D3D12_CPU_DESCRIPTOR_HANDLE (렌더링 출력용)
    └─ srv        → descriptor_index (bindless 셰이더 접근용)  ← 신규
```

### CreateSwapChain (변경 후)

```cpp
DXGI_SWAP_CHAIN_DESC1 swapChainDesc = {};
swapChainDesc.BufferUsage =
    DXGI_USAGE_RENDER_TARGET_OUTPUT | DXGI_USAGE_SHADER_INPUT;  // SRV 허용 추가
```

버퍼 초기화:
```cpp
textures.resize(buffer_count);
for (int i = 0; i < buffer_count; i++)
{
    auto state = std::make_shared<Texture_DX12>();

    // 1. Resource
    swapChain->GetBuffer(i, PPV_ARGS(state->resource));

    // 2. RTV (기존과 동일)
    state->rtv = allocationhandler->descriptors_rtv.allocate();
    device->CreateRenderTargetView(state->resource.Get(), &rtvDesc, state->rtv);

    // 3. SRV (신규)
    D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
    srvDesc.Format = format;
    srvDesc.ViewDimension = D3D12_SRV_DIMENSION_TEXTURE2D;
    srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
    srvDesc.Texture2D.MipLevels = 1;
    // bindless descriptor heap에 등록
    state->srv = allocationhandler->descriptors_res.allocate();
    device->CreateShaderResourceView(state->resource.Get(), &srvDesc, state->srv);

    textures[i].internal_state = state;
}
```

### GetBackBuffer() (변경 후)

```cpp
const Texture& GetBackBuffer(const SwapChain* swapchain)
{
    auto internal = to_internal(swapchain);
    return internal->textures[internal->GetBufferIndex()];
    // 힙 할당 0. 이미 존재하는 Texture 참조만 반환.
    // SRV 있음 → GetDescriptorIndex()로 bindless index 조회 가능
}
```

---

## 전/후 비교 요약

| 항목 | 변경 전 | 변경 후 |
|------|---------|---------|
| 내부 데이터 | `backBuffers[]` + `backbufferRTV[]` (분리) | `textures[]` (통합 Texture) |
| SRV 존재 | ❌ 없음 | ✅ 버퍼당 SRV 생성 |
| bindless descriptor index | ❌ 없음 | ✅ 셰이더에서 직접 접근 가능 |
| `GetBackBuffer()` 비용 | 매 호출마다 `make_shared` 힙 할당 | 기존 객체 참조 반환 (할당 0) |
| DXGI_USAGE 플래그 | `RENDER_TARGET_OUTPUT`만 | `+ SHADER_INPUT` 추가 |
| `dummyTexture` | 있음 (임시 반환용) | 제거됨 |
| 셰이더에서 backbuffer 읽기 | 불가 | 가능 (SRV 바인딩) |
| 코드 일관성 | 스왑체인 버퍼가 특수 취급 | 일반 Texture와 동일하게 관리 |

---

## 왜 이 구조가 더 나은가

### 1. 단일 책임 — 통합 Texture 관리

변경 전에는 "스왑체인 버퍼"와 "일반 Texture"가 서로 다른 방식으로 관리됐다.
변경 후에는 둘 다 `Texture_DX12` 객체로 통일되어, 후처리 패스가 swapchain backbuffer와
일반 RT를 구분 없이 다룰 수 있다.

### 2. 셰이더-readable 백버퍼의 실제 사용처

```
[프레임 렌더링 완료 → swapchain 백버퍼에 결과 있음]
         ↓
    Barrier: RENDER_TARGET → SHADER_RESOURCE
         ↓
[후처리 셰이더: backbuffer를 SRV로 읽어 추가 효과 적용]
  예: 화면 공간 굴절, 유리 효과, 화면 스크린샷 처리
         ↓
    Barrier: SHADER_RESOURCE → PRESENT
         ↓
    Present()
```

변경 전에는 이 흐름을 위해 별도 중간 RT에 backbuffer 내용을 복사해야 했다.
변경 후에는 backbuffer를 SRV로 직접 사용하므로 복사 비용이 없다.

### 3. `dummyTexture` 제거

변경 전 `dummyTexture`는 `GetBackBuffer()` 반환값을 위한 임시 컨테이너였다.
매번 새로운 `Texture_DX12` 객체를 만들다 보니 resource lifetime 관리가 애매했다.
변경 후에는 `textures[]`에 실제 완전한 Texture가 있으므로 이 문제가 사라진다.
