# Bindless Resources와 지연 삭제 (Deferred Deletion)

작성 중인 문서 `appendix_double_buffering.md`의 `AllocationHandler::Update()` 부분과 관련하여, **"Bindless(바인드리스)"란 무엇이며 왜 지연 삭제(Deferred Deletion)가 필요한지** 현대 그래픽스(DX12/Vulkan) 관점에서 정리한 문서입니다.

---

## 1. Bindless (바인드리스) 리소스란?

과거(DX11 이하)와 현대(DX12/Vulkan)의 리소스(텍스처, 버퍼 등) 처리 방식의 가장 큰 차이점 중 하나가 **Bindless 아키텍처**입니다.

### 1.1 기존 방식 (Bindful)의 문제점
과거에는 그리기(Draw) 콜을 하기 전에 매번 셰이더가 쓸 텍스처나 버퍼를 특정 슬롯(Slot)에 "바인딩(Binding)"해주어야 했습니다.
```cpp
// 매번 바꿀 때마다 CPU에서 바인딩 호출
deviceContext->PSSetShaderResources(0, 1, &textureA);
Draw();
deviceContext->PSSetShaderResources(0, 1, &textureB);
Draw();
```
* 문제: 텍스처가 1000개 있고 각각 다른 재질이라면, 바인딩 함수를 1000번 호출해야 하므로 **CPU 오버헤드(병목)**가 엄청나게 커집니다.

### 1.2 현대 방식: Bindless (바인드리스)
DX12에서는 **Descriptor Heap (디스크립터 힙)**이라는 거대한 배열을 사용합니다.
모든 텍스처와 버퍼의 디스크립터(포인터 같은 개념)를 이 거대한 배열 하나에 전부 몰아넣습니다. 그리고 셰이더(Shader)에는 텍스처 자체가 아니라 **"텍스처가 있는 배열의 인덱스 번호(정수)"**만 넘겨줍니다.

```cpp
// C++: "이번에 그릴 물체는 디스크립터 힙의 5번, 12번 인덱스를 씁니다"
// (바운딩 함수 호출이 획기적으로 줄어듦)
PushConstants(5, 12);
Draw();
```

```hlsl
// HLSL (Shader):
Texture2D bindless_textures[] : register(t0, space0);
// 셰이더 내부에서 인덱스로 직접 접근
float4 color = bindless_textures[material.diffuse_texture_index].Sample(...);
```

이것을 **Bindless(바인드리스)**라고 부릅니다. 특정 슬롯에 리소스를 제한적으로 묶어두는(Bind) 과정이 없어졌기 때문입니다. 덕분에 한 번의 Draw Call로 수많은 재질과 텍스처를 동시에 처리할 수 있는 강력한 장점이 있습니다.

---

## 2. 지연 삭제 (Deferred Deletion)가 필요한 이유

Bindless 환경과 더불어 **더블 버퍼링(Double Buffering)**(또는 트리플 버퍼링)이 적용되면, 리소스를 갱신하거나 삭제할 때 아주 골치 아픈 문제가 발생합니다.

### 2.1 최악의 시나리오 (Crash 발생)
1. **Frame N**: CPU가 "몬스터 A"를 그리는 커맨드를 만들어 GPU 큐에 넣습니다. (이 몬스터는 Bindless 배열의 `50번` 텍스처를 씁니다.)
2. **Frame N 렌더링 중**: GPU가 프레임 N을 열심히 그리고 있습니다.
3. CPU는 GPU를 기다리지 않고 **Frame N+1**을 준비합니다. 이때 몬스터 A가 죽어서 화면에서 제거됩니다.
4. CPU: *"몬스터 A가 죽었으니 50번 텍스처를 메모리에서 삭제(또는 재사용)하자!"* 라고 결합을 끊고 지워버립니다.
5. **GPU**: Frame N에서 50번 텍스처를 읽으려고 접근했는데, CPU가 이미 지워버렸습니다! 💥 **GPU Page Fault (Device Removed 에러 / 엔진 크래시)**

### 2.2 해결책: GPU가 다 쓸 때까지 기다렸다가 지운다
CPU가 리소스를 지우고 싶어도, **과거 프레임에서 GPU가 아직 그 리소스를 사용 중일 수 있기 때문에 곧바로 지우면 안 됩니다.** GPU가 해당 작업(Frame N)을 완전히 끝냈다는 것이 보장되는 시점에 지워야 합니다.

이것이 **지연 삭제 (Deferred Deletion)** 개념입니다. 

---

## 3. VizMotive 엔진의 `AllocationHandler::Update()` 분석

실제 VizMotive 엔진 코드(`GraphicsDevice_DX12.h` 파일의 `AllocationHandler::Update()` 함수)를 보면 이 원리가 그대로 구현되어 있습니다.

### 3.1 삭제 대기열 (Destroyer Queue)
```cpp
std::deque<std::pair<Microsoft::WRL::ComPtr<ID3D12Resource>, uint64_t>> destroyer_resources;
std::deque<std::pair<int, uint64_t>> destroyer_bindless_res;
```
엔진에서 리소스(버퍼/텍스처)를 삭제하려고 하면 즉시 메모리를 해제하지 않습니다. 대신 `[지울 리소스, 삭제를 요청한 당시의 FRAMECOUNT]` 쌍을 만들어 대기열(`deque`)에 밀어 넣습니다.

* `destroyer_bindless_res`: Bindless 배열에서 썼던 인덱스를 담아두는 대기열.

### 3.2 안전한 삭제 시점 판단 로직
매 프레임마다 호출되는 `Update()` 함수의 핵심 로직입니다:

```cpp
// Deferred destroy of resources that the GPU is already finished with:
void Update(uint64_t FRAMECOUNT, uint32_t BUFFERCOUNT)
{
    // ...
    while (!destroyer_bindless_res.empty() && 
           destroyer_bindless_res.front().second + BUFFERCOUNT < FRAMECOUNT)
    {
        int index = destroyer_bindless_res.front().first;
        destroyer_bindless_res.pop_front();
        free_bindless_res.push_back(index); // 이제야 비로소 인덱스 재사용 가능
    }
}
```

**설명:**
- `destroyer_bindless_res.front().second` : 해당 리소스를 삭제해달라고 큐에 넣었던 시점의 프레임 카운트 (예: `Frame N`)
- `BUFFERCOUNT`: 현재 엔진에서 사용하는 버퍼 개수 (더블 버퍼링이면 `2`)
- `FRAMECOUNT`: 현재 CPU가 돌고 있는 프레임

즉, **`삭제 요청 시점(N) + BUFFERCOUNT(2) < 현재 프레임 카운트`** 가 될 때 비로소 큐에서 빼내어 실제로 리소스를 날리거나 Bindless 인덱스를 재활용(free_...) 리스트로 보냅니다.

> *"이 리소스를 지우라고 한 지 최소 BUFFERCOUNT 프레임이 지났다면, GPU도 무조건 예전 프레임 작업을 다 끝냈을 테니까 이제는 진짜 지워도 안전하다!"*

이전 프레임 펜스 문서를 다루면서 배웠듯이, CPU는 GPU를 최대 `BUFFERCOUNT - 1` 프레임까지만 앞설 수 있도록 `CPU wait`로 동기화가 맞춰져 있습니다. 따라서 `BUFFERCOUNT` 이상 차이가 난다는 것은 **GPU가 관련된 모든 그리기 작업을 명확하게 종료했음**을 수학적으로 보장합니다.

---

## 4. 요약
1. **Bindless 환경**: 수많은 리소스를 거대한 배열(Descriptor Heap)에 다 넣어두고 인덱스로만 접근하여 CPU 오버헤드를 극적으로 줄이는 현대적 렌더링 방식.
2. **충돌 위험**: 더블(멀티) 버퍼링을 쓸 때, CPU가 텍스처나 버퍼를 갱신/지워버리면, 뒤따라오는 GPU(과거 프레임을 그리는 중)가 허공을 참조하다가 뻗어버릴 수 있음.
3. **지연 처리 (Deferred)**: 그래서 버퍼 정보나 리소스를 즉각 지우는 대신, **"Queue에 현재 프레임 넘버를 달아서 킵해두었다가, GPU가 확실히 다 쓰고 나서 (프레임 넘버 + BUFFERCOUNT) 비로소 진짜 삭제하거나 재사용"**합니다.

이것이 `appendix_double_buffering.md`의 L197 주변에 있던 "리소스 지연 삭제"와 "Bindless buffer updates"의 핵심 원리입니다.
