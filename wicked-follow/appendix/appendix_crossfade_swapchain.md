# Appendix: WickedEngine CrossFade — Backbuffer SRV 실제 사용 여부 검증

> 이 문서를 읽기 전 권장 선행 자료:
> - [appendix_swapchain_srv_comparison.md](appendix_swapchain_srv_comparison.md) — SRV 추가 전/후 구조
> - [appendix_vizmotive_swapchain.md](appendix_vizmotive_swapchain.md) — VizMotive 포스트프로세싱 흐름

---

## CrossFade란

CrossFade는 씬 전환 시 부드럽게 블렌딩하는 화면 전환 효과다.
두 씬을 동시에 렌더링하는 방식이 아니라 **스크린샷 오버레이** 방식으로 구현된다.

---

## 직관적 이해 vs 실제 구현

### 직관적 이해 (영상 편집의 crossfade)

```
Old scene: opacity 1.0 ──────────────▶ 0.0
New scene: opacity 0.0 ──────────────▶ 1.0
           두 씬을 동시에 렌더링
```

GPU 비용: 2× (두 씬 모두 매 프레임 렌더)

### WickedEngine 실제 구현 (스크린샷 오버레이)

```
전환 직전 1프레임:
  [Old scene 렌더] → backbuffer
  [backbuffer 캡처] → crossFadeTexture (스크린샷)

전환 후 매 프레임:
  [New scene 렌더] → backbuffer         (실제 렌더)
  [crossFadeTexture를 위에 덮어 그림]    (opacity 1.0 → 0.0)
```

GPU 비용: 1× + 풀스크린 quad 1장 (매우 저렴)

**결과**: 사용자 눈에는 두 씬이 블렌딩되는 것처럼 보이지만,
Old scene은 실제로 렌더링되지 않고 스크린샷이 점점 투명해지는 것이다.

트레이드오프: Old scene의 애니메이션(캐릭터 움직임, 파티클)이 전환 중 멈춰 보인다.
씬 전환(로딩 화면 → 게임 등)에서는 문제없다.

---

## FadeManager 상태 머신

```
FADE_IN  → crossFadeTexture opacity 0.0 → 1.0  (오래된 화면이 점점 진하게 오버레이)
FADE_MID → opacity = 1.0                       (이 시점에 씬 교체 실행)
FADE_OUT → crossFadeTexture opacity 1.0 → 0.0  (오버레이가 사라지며 새 씬 드러남)
FADE_FINISHED
```

```cpp
// wiFadeManager.h
struct FadeManager {
    float opacity = 0;
    float timer = 0;
    float targetFadeTimeInSeconds = 1.0f;
    enum FadeType { FadeToColor, CrossFade };
    enum FADE_STATE { FADE_IN, FADE_MID, FADE_OUT, FADE_FINISHED };
    wi::graphics::Texture crossFadeTexture;         // 캡처된 스크린샷
    bool crossFadeTextureSaveRequired = false;      // 캡처 요청 플래그
    std::function<void()> onFade = [] {};           // 씬 교체 콜백
};
```

---

## CrossFade 트리거 위치

| 위치 | 조건 |
|------|------|
| `Editor.cpp:354` | 에디터 초기화 완료 (0.5초) |
| `Editor.cpp:886` | 새 Terrain 생성 (1초) |
| `wiLoadingScreen.cpp:45` | 로딩 완료 후 씬 전환 |
| Lua 스크립트 | `application:CrossFade(duration)` |

---

## 실제 코드 흐름 (wiApplication.cpp)

### 1단계: 캡처 (crossFadeTextureSaveRequired = true인 프레임)

```cpp
// wiApplication.cpp:334
if (fadeManager.crossFadeTextureSaveRequired)
{
    Texture backbuffer = rendertargetPreHDR10.IsValid()
        ? rendertargetPreHDR10
        : graphicsDevice->GetBackBuffer(&swapChain);

    // crossFadeTexture 크기/포맷 맞추기
    if (crossFadeTexture.desc.width != backbuffer.desc.width || ...)
    {
        TextureDesc desc = backbuffer.desc;
        desc.bind_flags = BindFlag::SHADER_RESOURCE;
        graphicsDevice->CreateTexture(&desc, nullptr, &fadeManager.crossFadeTexture);
    }

    // 상태 전환: 현재 레이아웃 → COPY_SRC
    PushBarrier(GPUBarrier::Image(&backbuffer, backbuffer.desc.layout, ResourceState::COPY_SRC));
    PushBarrier(GPUBarrier::Image(&crossFadeTexture, ..., ResourceState::COPY_DST));
    FlushBarriers(cmd);

    // 복사 (SRV 아님 — GPU 메모리 직접 복사)
    graphicsDevice->CopyResource(&fadeManager.crossFadeTexture, &backbuffer, cmd);

    // 상태 복원
    PushBarrier(GPUBarrier::Image(&crossFadeTexture, ResourceState::COPY_DST, ...));
    PushBarrier(GPUBarrier::Image(&backbuffer, ResourceState::COPY_SRC, ...));
    FlushBarriers(cmd);

    fadeManager.crossFadeTextureSaveRequired = false;
}
```

### 2단계: 오버레이 렌더링 (매 프레임, 전환 중)

```cpp
// wiApplication.cpp:450
if (fadeManager.IsActive())
{
    wi::image::Params fx;
    fx.enableFullScreen();
    fx.opacity = fadeManager.opacity;  // 1.0 → 0.0으로 감소

    if (fadeManager.type == FadeManager::FadeType::CrossFade)
    {
        // crossFadeTexture를 SRV로 읽어 풀스크린 quad에 그림
        wi::image::Draw(&fadeManager.crossFadeTexture, fx, cmd);
    }
}
```

---

## Backbuffer SRV가 실제로 사용되는가?

```
캡처 단계: CopyResource(&crossFadeTexture, &backbuffer)
           → backbuffer는 COPY_SRC 상태
           → SRV 바인딩 없음 ❌

오버레이 단계: wi::image::Draw(&crossFadeTexture, ...)
              → crossFadeTexture의 SRV를 사용
              → backbuffer SRV는 여기서도 사용 안 됨 ❌
```

**결론: CrossFade는 backbuffer SRV를 사용하지 않는다.**

backbuffer는 `CopyResource`의 소스(COPY_SRC)로만 쓰인다.
셰이더에서 읽히는 것은 복사된 `crossFadeTexture`이며, 이것은 별도로 생성된 텍스처다.

---

## 왜 이 검증이 중요한가

`appendix_swapchain_srv_comparison.md`에 SRV 추가의 이점으로
"스크린샷 효율 개선, CrossFade 활용"이 언급되어 있었다.

실제 확인 결과:
- CrossFade: `CopyResource` 사용 → SRV 불필요
- 스크린샷: readback 복사 → SRV 불필요
- HDR10 blit: `rendertargetPreHDR10` 사용 → backbuffer SRV 불필요

**backbuffer SRV가 실제로 사용되는 곳은 WickedEngine 코드베이스 내에 없다.**
SRV는 구조적 완전성을 위해 추가된 인프라이며, 미래의 화면-읽기 효과를 위한 준비 상태다.

→ VizMotive는 SRV 없이 구조 정리만 적용: `apply_texture.md 변경 3` 참고
