# Part 5: 텍스처 전체 흐름 — 이미지에서 화면까지

> **선행 권장**: [Part 4: 리소스 관리](part4_resource_management.md) — 힙, 버퍼, 디스크립터 기본

이 문서 하나로 다음 질문에 완전히 답한다:

> **"PNG 파일 하나가 어떻게 GPU에 올라가서 화면에 렌더링되는가?"**

---

## 5.1 전체 흐름 한 눈에 보기

```
[디스크]          [CPU 메모리]         [GPU 메모리]               [화면]
  │                   │                    │
  ▼                   │                    │
PNG / DDS 파일        │                    │
  │                   │                    │
  │  파일 읽기 +       │                    │
  │  디코딩           │                    │
  ├──────────────▶  픽셀 데이터           │
  │               (RGBA 배열)             │
  │                   │                    │
  │           Upload heap에 복사          │
  │               ├──────────────────▶  Upload Heap (선형)
  │               │                    스테이징 버퍼
  │               │                       │
  │               │     CopyTextureRegion │
  │               │     (GPU 복사 명령)   │
  │               │                       ├───────▶ DEFAULT Heap (타일드)
  │               │                       │        ID3D12Resource (텍스처)
  │               │                       │
  │               │              Resource Barrier
  │               │          (COPY_DEST → SHADER_RESOURCE)
  │               │                       │
  │               │                  Descriptor 생성
  │               │                  (SRV 핸들)
  │               │                       │
  │               │              셰이더에서 SRV로 샘플링
  │               │                       │
  │               │               렌더 결과 → SwapChain 백버퍼
  │               │                                    │
  │               │                               Present()
  │               │                                    ▼
  │               │                               [화면 출력]
```

각 단계를 하나씩 뜯어본다.

---

## 5.2 파일 디코딩 — CPU 메모리에 픽셀 올리기

### PNG / JPG 같은 파일은 "압축된" 이미지다

```
PNG 파일 (디스크):       → 압축된 형태, GPU가 직접 못 씀
RGBA 픽셀 배열 (CPU):    → 압축 해제된 원본 데이터
```

DirectXTex, stb_image 같은 라이브러리로 디코딩하면 CPU 메모리에
`width × height × bytes_per_pixel` 크기의 선형 배열이 생긴다.

```cpp
// stb_image 예시
int width, height, channels;
uint8_t* pixels = stbi_load("texture.png", &width, &height, &channels, 4);
// → pixels[0..width*height*4] = RGBA 픽셀 데이터 (선형 배열)
```

### DDS 파일은 특별하다

DDS(DirectDraw Surface) 포맷은 GPU가 바로 쓸 수 있는 형태로 저장된 파일이다.

```
DDS 파일:
  ├─ 헤더 (포맷, 크기, mip 개수 등)
  ├─ Mip 0 데이터 (가장 큰 해상도)
  ├─ Mip 1 데이터 (절반 크기)
  ├─ Mip 2 데이터 (1/4 크기)
  └─ ...
```

BC 압축(BC1~BC7)도 DDS에 그대로 들어있다. 디코딩 없이 GPU에 바로 올릴 수 있다.

---

## 5.3 Mipmap — 거리별 해상도 최적화

### Mipmap이 무엇인가

원본 텍스처의 축소 버전들을 미리 만들어 함께 저장한 것.

```
원본 (Mip 0): 1024 × 1024
Mip 1:         512 ×  512
Mip 2:         256 ×  256
Mip 3:         128 ×  128
...
Mip 10:          1 ×    1
```

### 왜 필요한가

```
[카메라]

  가까운 오브젝트:
  ┌──────────────────────┐
  │                      │   → 화면에 크게 표시
  │   1024×1024 텍스처   │   → Mip 0 사용 (고해상도)
  │                      │
  └──────────────────────┘

  멀리 있는 오브젝트:
  ┌───┐
  │텍│   → 화면에 아주 작게 표시 (10×10 픽셀)
  └───┘   → Mip 0 쓰면? 1024×1024에서 10×10으로 다운샘플
           = 많은 픽셀 버림 → 모아레/깜빡임(aliasing) 발생!
           → Mip 4 (64×64) 사용 = 딱 맞는 해상도
```

GPU가 자동으로 적절한 mip 레벨을 선택한다(LOD: Level of Detail).

### 총 메모리는 얼마나 늘어나나

```
1024×1024 원본:  1,048,576 픽셀
512×512  Mip 1:    262,144 픽셀
256×256  Mip 2:     65,536 픽셀
...모든 mip 합: ≈ 1,048,576 × 4/3 = 1,398,101 픽셀
```

원본의 약 1.33배. 미리 저장해두는 비용은 작고, 얻는 품질/성능은 크다.

### mip_levels = 0의 의미

DX12에서 `MipLevels = 0`으로 리소스를 생성하면,
드라이버가 가능한 최대 mip 수를 자동으로 계산한다.

```
1024×1024 → 최대 mip = log2(1024) + 1 = 11 레벨 (1024, 512, ..., 1)
```

---

## 5.4 GPU 텍스처 메모리 레이아웃 — 선형 vs 타일드

### CPU 메모리: 선형(Linear) 레이아웃

```
픽셀[0,0] 픽셀[1,0] 픽셀[2,0] ... 픽셀[W-1,0]  ← Row 0
픽셀[0,1] 픽셀[1,1] ...                           ← Row 1
...
```

Row-major 순서로 연속된 배열. CPU가 읽기 쉬운 형태.

### GPU 메모리: 타일드(Tiled) 레이아웃

GPU는 텍스처를 **타일(예: 64×64픽셀 블록) 단위**로 메모리에 저장한다.

```
[타일 0]  [타일 1]  [타일 2]
[타일 3]  [타일 4]  [타일 5]
...

각 타일 내부는 GPU에 최적화된 Z-order curve(Morton order) 등으로 배치
```

이유: 셰이더가 텍스처를 샘플링할 때 주변 픽셀을 함께 읽는 경우가 많다.
타일드 레이아웃은 **공간적 지역성(spatial locality)**이 좋아서 캐시 효율이 높다.

```
선형 레이아웃에서 (x, y) 근방 샘플링:
  → (x+1, y)는 바로 옆 메모리
  → (x, y+1)는 Row 하나 건너뜀 → 캐시 미스!

타일드 레이아웃에서:
  → 주변 픽셀이 같은 타일 내에 있음 → 캐시 히트
```

### DX12에서의 표현

```
DEFAULT Heap + D3D12_TEXTURE_LAYOUT_UNKNOWN  → 타일드 (GPU 최적화)
UPLOAD/READBACK Heap                          → 선형 (CPU 접근 가능)
```

`TEXTURE_LAYOUT_UNKNOWN`이란 "드라이버가 알아서 최적화해"라는 의미다.

---

## 5.5 Footprint — UPLOAD heap의 선형 텍스처 표현

### 왜 Footprint가 필요한가

GPU에 텍스처를 올리려면 Upload heap에 먼저 데이터를 올려야 한다.
그런데 Upload heap은 버퍼(선형 배열)다. 2D 이미지를 선형 버퍼에 담을 때
정확한 레이아웃 정보가 필요하다. 이것이 **Footprint**다.

```
UPLOAD 버퍼 (선형):
│ 오프셋 0
│ ├─ Row 0: [픽셀 0][픽셀 1]...[픽셀 W-1][..패딩..]  ← RowPitch (256 정렬)
│ ├─ Row 1: [픽셀 0][픽셀 1]...[픽셀 W-1][..패딩..]
│ ├─ ...
│ └─ Row H-1: ...
│ 오프셋 = Offset + RowPitch * H
```

### Footprint 구조

```cpp
struct D3D12_PLACED_SUBRESOURCE_FOOTPRINT {
    UINT64 Offset;         // 버퍼 내 이 서브리소스의 시작 위치
    struct {
        DXGI_FORMAT Format;
        UINT Width;
        UINT Height;
        UINT Depth;
        UINT RowPitch;     // 한 Row의 바이트 수 (256의 배수로 정렬)
    } Footprint;
};
```

### RowPitch는 왜 256 정렬인가

DX12 스펙: Upload heap에서 텍스처 Row는 반드시 `D3D12_TEXTURE_DATA_PITCH_ALIGNMENT = 256` 바이트 정렬.

```
예: width = 100, RGBA8 = 4 bytes/pixel
실제 데이터 크기: 100 × 4 = 400 bytes
RowPitch:         ceil(400 / 256) × 256 = 512 bytes  ← 112 bytes 패딩
```

패딩이 있기 때문에 CPU 픽셀 배열을 그냥 memcpy하면 안 된다.
Row 하나씩 복사해야 한다.

```cpp
// Row별 복사
void* mapped;
uploadBuffer->Map(0, nullptr, &mapped);
const uint8_t* src = pixelData;
uint8_t* dst = static_cast<uint8_t*>(mapped) + footprint.Offset;
for (uint32_t row = 0; row < height; ++row) {
    memcpy(dst + row * footprint.Footprint.RowPitch,
           src + row * width * 4,       // 실제 데이터 크기
           width * 4);                  // 패딩 없이 복사
}
uploadBuffer->Unmap(0, nullptr);
```

### Footprint 계산: GetCopyableFootprints

DX12가 직접 계산해준다.

```cpp
UINT64 totalSize;
std::vector<D3D12_PLACED_SUBRESOURCE_FOOTPRINT> footprints(mipLevels);
std::vector<UINT64> rowSizes(mipLevels);
std::vector<UINT> numRows(mipLevels);

device->GetCopyableFootprints(
    &resourceDesc,   // 텍스처 디스크립션
    0,               // 첫 번째 서브리소스
    mipLevels,       // 서브리소스 개수
    0,               // 버퍼 내 시작 오프셋
    footprints.data(),
    numRows.data(),
    rowSizes.data(),
    &totalSize       // 전체 필요 버퍼 크기
);
// → totalSize 크기의 Upload buffer 생성 후 데이터 채우기
```

---

## 5.6 Subresource — DEFAULT heap 텍스처의 특정 mip/slice/plane 지정

### Subresource Index란

GPU 텍스처(DEFAULT heap)의 특정 "조각"을 지정하는 정수 인덱스.

```
subresource_index = mip + (slice × mip_levels) + (plane × mip_levels × array_size)
```

**mip**: 0 = 원본(최고해상도), 1 = 절반, 2 = 1/4 ...
**slice**: 텍스처 배열의 몇 번째 (단일 텍스처면 0)
**plane**: 대부분 0. 깊이+스텐실처럼 두 평면이 있는 포맷만 1이 있음

예시:

```
1024×1024 텍스처, mip_levels=3, array_size=1:

subresource 0: mip 0, slice 0  (1024×1024 원본)
subresource 1: mip 1, slice 0  (512×512)
subresource 2: mip 2, slice 0  (256×256)

텍스처 배열 (mip_levels=3, array_size=2):

subresource 0: mip 0, slice 0
subresource 1: mip 1, slice 0
subresource 2: mip 2, slice 0
subresource 3: mip 0, slice 1
subresource 4: mip 1, slice 1
subresource 5: mip 2, slice 1
총 6개
```

D3D12CalcSubresource(mip, slice, plane, mipLevels, arraySize) 헬퍼 함수로 계산한다.

### Subresource vs Footprint 복사

```
GPU 텍스처 (DEFAULT) 복사 시:
  → SubresourceIndex로 mip/slice 지정

CPU 텍스처 (UPLOAD/READBACK) 복사 시:
  → Footprint로 버퍼 내 위치/크기 지정

CopyTextureRegion에서:
  DEFAULT ↔ DEFAULT:  SubresourceIndex ↔ SubresourceIndex
  DEFAULT ↔ UPLOAD:   SubresourceIndex ↔ Footprint
```

---

## 5.7 BC (Block Compressed) 텍스처

### 왜 압축하나

```
1024×1024 RGBA8 텍스처:  1024 × 1024 × 4 = 4 MB
1024×1024 BC7 텍스처:    1024/4 × 1024/4 × 16 = 1 MB  ← 4배 압축
```

GPU에서 실시간으로 압축 해제하므로 셰이더는 일반 텍스처처럼 샘플링한다.

### BC 포맷 종류

| 포맷 | 압축률 | 용도 |
|------|--------|------|
| BC1 (DXT1) | 8:1 | 불투명 컬러 (알파 없거나 1비트) |
| BC2 (DXT3) | 4:1 | 날카로운 알파 (거의 안 씀) |
| BC3 (DXT5) | 4:1 | 부드러운 알파 |
| BC4 | 2:1 | 단일 채널 (높이맵, AO 맵) |
| BC5 | 2:1 | 2채널 (노말맵 XY) |
| BC6H | 4:1 | HDR (부동소수점) |
| BC7 | 4:1 | 고품질 컬러+알파 |

### BC 텍스처의 블록 구조

BC 텍스처는 4×4 픽셀을 하나의 블록으로 묶어 압축한다.

```
원본 이미지 (8×4 픽셀):
┌────┬────┐
│블록│블록│  ← 4×4 블록 2개 = 8×4 픽셀
└────┴────┘

BC1 기준: 4×4 블록 = 8 bytes
→ 8×4 이미지 = 2 블록 × 8 bytes = 16 bytes
→ RGBA8로는 8×4×4 = 128 bytes → 8배 압축
```

### 블록 수 계산 — Ceiling Division이 왜 필요한가

width가 4의 배수가 아닌 경우:

```
width = 10 pixels (BC 포맷, block_size = 4)

올바른 계산:
  블록 수 = ceil(10 / 4) = 3 블록  (4+4+2 픽셀 → 마지막 블록은 2픽셀만 유효)

단순 나눗셈:
  블록 수 = 10 / 4 = 2 블록  ← 마지막 2픽셀 날아감!
```

올바른 공식: `blocks = (pixels + block_size - 1) / block_size`

이것이 C++에서 "ceiling division"이다.

---

## 5.8 업로드 전체 코드 흐름

파일 로드부터 GPU 텍스처 완성까지 실제 코드 순서:

```cpp
// ── 1. 파일 디코딩 (CPU) ─────────────────────────────────
int width, height;
std::vector<uint8_t> pixelData = LoadImage("texture.png", width, height);
// pixelData: RGBA8, width × height × 4 bytes, 선형 배열

// ── 2. GPU 텍스처 디스크립션 ─────────────────────────────
D3D12_RESOURCE_DESC texDesc = {};
texDesc.Dimension         = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
texDesc.Width             = width;
texDesc.Height            = height;
texDesc.DepthOrArraySize  = 1;
texDesc.MipLevels         = 1;           // 밉맵 없이 원본만
texDesc.Format            = DXGI_FORMAT_R8G8B8A8_UNORM;
texDesc.SampleDesc.Count  = 1;
texDesc.Layout            = D3D12_TEXTURE_LAYOUT_UNKNOWN;  // 타일드 레이아웃
texDesc.Flags             = D3D12_RESOURCE_FLAG_NONE;

// ── 3. DEFAULT Heap에 GPU 텍스처 생성 ────────────────────
// 처음엔 COPY_DEST 상태 (Upload에서 받을 준비)
D3D12_HEAP_PROPERTIES defaultHeap = { D3D12_HEAP_TYPE_DEFAULT };
device->CreateCommittedResource(
    &defaultHeap, D3D12_HEAP_FLAG_NONE,
    &texDesc,
    D3D12_RESOURCE_STATE_COPY_DEST,   // ← 복사 받을 준비 상태
    nullptr,
    IID_PPV_ARGS(&gpuTexture)
);

// ── 4. Footprint 계산 ────────────────────────────────────
D3D12_PLACED_SUBRESOURCE_FOOTPRINT footprint;
UINT64 uploadSize;
device->GetCopyableFootprints(&texDesc, 0, 1, 0, &footprint, nullptr, nullptr, &uploadSize);

// ── 5. Upload Heap에 스테이징 버퍼 생성 ──────────────────
D3D12_RESOURCE_DESC bufDesc = {};
bufDesc.Dimension = D3D12_RESOURCE_DIMENSION_BUFFER;
bufDesc.Width     = uploadSize;
bufDesc.Height    = 1; bufDesc.DepthOrArraySize = 1;
bufDesc.MipLevels = 1; bufDesc.SampleDesc.Count = 1;
bufDesc.Layout    = D3D12_TEXTURE_LAYOUT_ROW_MAJOR;  // 버퍼는 선형

D3D12_HEAP_PROPERTIES uploadHeap = { D3D12_HEAP_TYPE_UPLOAD };
device->CreateCommittedResource(
    &uploadHeap, D3D12_HEAP_FLAG_NONE,
    &bufDesc,
    D3D12_RESOURCE_STATE_GENERIC_READ,  // Upload는 항상 이 상태
    nullptr,
    IID_PPV_ARGS(&uploadBuffer)
);

// ── 6. 스테이징 버퍼에 픽셀 데이터 복사 ─────────────────
void* mapped;
uploadBuffer->Map(0, nullptr, &mapped);
uint8_t* dst = static_cast<uint8_t*>(mapped) + footprint.Offset;
for (int row = 0; row < height; ++row) {
    memcpy(
        dst + row * footprint.Footprint.RowPitch,  // RowPitch 정렬 고려
        pixelData.data() + row * width * 4,        // 원본 데이터
        width * 4
    );
}
uploadBuffer->Unmap(0, nullptr);

// ── 7. GPU 복사 명령 기록 ─────────────────────────────────
D3D12_TEXTURE_COPY_LOCATION src = {};
src.pResource       = uploadBuffer;
src.Type            = D3D12_TEXTURE_COPY_TYPE_PLACED_FOOTPRINT;
src.PlacedFootprint = footprint;

D3D12_TEXTURE_COPY_LOCATION dst_loc = {};
dst_loc.pResource        = gpuTexture;
dst_loc.Type             = D3D12_TEXTURE_COPY_TYPE_SUBRESOURCE_INDEX;
dst_loc.SubresourceIndex = 0;  // mip 0, slice 0

commandList->CopyTextureRegion(&dst_loc, 0, 0, 0, &src, nullptr);

// ── 8. 상태 전이: COPY_DEST → SHADER_RESOURCE ────────────
D3D12_RESOURCE_BARRIER barrier = {};
barrier.Type                   = D3D12_RESOURCE_BARRIER_TYPE_TRANSITION;
barrier.Transition.pResource   = gpuTexture;
barrier.Transition.StateBefore = D3D12_RESOURCE_STATE_COPY_DEST;
barrier.Transition.StateAfter  = D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE;
barrier.Transition.Subresource = D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES;
commandList->ResourceBarrier(1, &barrier);

// ── 9. SRV 생성 ───────────────────────────────────────────
D3D12_SHADER_RESOURCE_VIEW_DESC srvDesc = {};
srvDesc.Shader4ComponentMapping = D3D12_DEFAULT_SHADER_4_COMPONENT_MAPPING;
srvDesc.Format                  = DXGI_FORMAT_R8G8B8A8_UNORM;
srvDesc.ViewDimension           = D3D12_SRV_DIMENSION_TEXTURE2D;
srvDesc.Texture2D.MipLevels     = 1;
device->CreateShaderResourceView(gpuTexture, &srvDesc, srvCpuHandle);

// ── 10. 커맨드 실행 후 업로드 버퍼 해제 가능 ─────────────
```

---

## 5.9 Descriptor(뷰) — 같은 텍스처를 여러 방식으로 쓰기

### Descriptor란

`ID3D12Resource`(실제 데이터)는 그냥 메모리 덩어리다.
이 메모리를 "어떤 용도로, 어떤 포맷으로 해석할 것인가"를 GPU에 알려주는 것이 Descriptor다.

```
ID3D12Resource (텍스처 메모리)
    │
    ├── SRV (Shader Resource View)
    │   셰이더에서 읽기: texture.Sample(sampler, uv)
    │
    ├── RTV (Render Target View)
    │   렌더 타겟으로 쓰기: OMSetRenderTargets()
    │
    ├── UAV (Unordered Access View)
    │   컴퓨트 셰이더에서 읽기/쓰기: RWTexture2D
    │
    └── DSV (Depth Stencil View)
        깊이/스텐실 버퍼로 쓰기: OMSetDepthStencil()
```

**같은 `ID3D12Resource`에 여러 Descriptor를 만들 수 있다.**

```cpp
// 렌더 타겟 텍스처: RTV + SRV 둘 다 생성
device->CreateRenderTargetView(renderTarget, nullptr, rtvHandle); // RTV
device->CreateShaderResourceView(renderTarget, &srvDesc, srvHandle); // SRV

// 사용 방법:
// 렌더 패스에서 RTV로 → 렌더링
// 다음 패스에서 SRV로 → 셰이더에서 읽기
```

### 각 Descriptor가 저장되는 Descriptor Heap

```
CBV/SRV/UAV Heap  ← SRV, CBV, UAV 저장 (셰이더에서 접근)
RTV Heap          ← RTV 저장 (파이프라인 설정용)
DSV Heap          ← DSV 저장 (파이프라인 설정용)
Sampler Heap      ← Sampler 저장
```

RTV/DSV는 "Shader Visible" 힙에 저장할 수 없다. 셰이더에서 직접 접근하는 게 아니라,
파이프라인 설정(`OMSetRenderTargets`)에서만 사용하기 때문이다.

---

## 5.10 Resource Barrier — 상태 전이

### 왜 상태를 명시적으로 바꿔야 하나

GPU는 파이프라인 내에서 동일한 리소스를 동시에 다른 용도로 접근하면 안 된다.
Barrier는 GPU에게 "이 리소스 쓰기 다 끝낸 뒤에 다음 작업 시작해"라고 알리는 신호다.

```
렌더 타겟에 그림 그리기 (RENDER_TARGET 상태)
          ↓
[ Barrier: RENDER_TARGET → PIXEL_SHADER_RESOURCE ]
  → GPU: 렌더링 다 끝내고 캐시 flush
  → GPU: 이제 셰이더에서 읽어도 됨
          ↓
셰이더에서 샘플링 (PIXEL_SHADER_RESOURCE 상태)
```

### 텍스처 업로드 흐름의 상태

```
CreateCommittedResource(..., COPY_DEST, ...)   → 처음 상태: COPY_DEST
CopyTextureRegion(dst, src)                    → Upload → DEFAULT 복사
ResourceBarrier(COPY_DEST → SHADER_RESOURCE)   → 이제 셰이더에서 읽을 수 있음
```

### 렌더 타겟 텍스처의 매 프레임 상태 변화

```
프레임 시작:
  PRESENT → RENDER_TARGET  (백버퍼: 이제 그릴 수 있음)

오프스크린 렌더링:
  RENDER_TARGET → PIXEL_SHADER_RESOURCE  (G-buffer: 이제 읽을 수 있음)

최종 렌더링 후:
  RENDER_TARGET → PRESENT  (백버퍼: 이제 화면에 표시)

Present()                  (화면에 출력)
```

---

## 5.11 SwapChain — 화면에 표시되는 특별한 텍스처

### SwapChain이란

화면에 표시되는 back buffer 텍스처를 관리하는 구조.
더블 버퍼링(buffer_count = 2)으로 이루어진다.

```
SwapChain:
  BackBuffer[0]  ←── GPU가 렌더링 중
  BackBuffer[1]  ←── 모니터에 표시 중

Present() 호출 후:
  BackBuffer[0]  ←── 모니터에 표시 중
  BackBuffer[1]  ←── GPU가 다음 프레임 렌더링 중
```

CPU와 모니터가 동시에 같은 버퍼를 보지 않도록 번갈아 사용한다.

### BackBuffer의 특별한 점

```cpp
// BackBuffer는 swapchain이 만들어서 준다 (직접 생성 X)
swapChain->GetBuffer(i, IID_PPV_ARGS(&backBuffer[i]));

// RTV만 생성하면 렌더 타겟으로 사용 가능
device->CreateRenderTargetView(backBuffer[i], nullptr, rtvHandle[i]);
```

BackBuffer는:
- `D3D12_RESOURCE_STATE_PRESENT` 상태로 시작/끝
- 렌더링 전: `PRESENT → RENDER_TARGET` 전이
- 렌더링 후: `RENDER_TARGET → PRESENT` 전이
- `Present()` 호출 후 화면에 출력

### Shader-readable SwapChain (DXGI_USAGE_SHADER_INPUT)

기본적으로 BackBuffer는 SRV를 만들 수 없다(shader에서 읽기 불가).
후처리 효과(Bloom, FXAA 등)를 위해 BackBuffer를 셰이더에서 읽으려면:

```cpp
// SwapChain 생성 시
swapChainDesc.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT
                          | DXGI_USAGE_SHADER_INPUT;  // ← 이 플래그 필요

// 그러면 SRV도 생성 가능
device->CreateShaderResourceView(backBuffer[i], &srvDesc, srvHandle[i]);
```

---

## 5.12 VizMotive 엔진에서의 흐름 요약

VizMotive의 `CreateTexture()` → `GetBackBuffer()` → `DeleteSubresources()` 흐름:

```
[앱 코드]                        [GraphicsDevice_DX12.cpp]
CreateTexture(desc, initData)
    │
    ├─ 1. Texture_DX12 생성       ← make_shared<Texture_DX12>()
    │
    ├─ 2. D3D12 리소스 생성       ← CreateCommittedResource (DEFAULT heap)
    │       COPY_DEST 상태
    │
    ├─ 3. 초기 데이터 있으면 업로드
    │       CopyAllocator.allocate()    ← Upload heap 스테이징 버퍼
    │       CopyTextureRegion()         ← GPU 복사 명령
    │       CopyAllocator.submit()      ← 즉시 실행 + CPU wait
    │
    ├─ 4. SRV 생성                ← CreateShaderResourceView
    │       (bind_flags에 SHADER_RESOURCE 포함 시)
    │
    ├─ 5. RTV 생성                ← CreateRenderTargetView
    │       (bind_flags에 RENDER_TARGET 포함 시)
    │
    └─ 6. DSV 생성                ← CreateDepthStencilView
            (bind_flags에 DEPTH_STENCIL 포함 시)

    → internal_state(= Texture_DX12)를 Texture.internal_state에 저장

[사용]
  commandList->SetGraphicsRootDescriptorTable(SRV)  ← 셰이더에서 샘플링

[삭제]
DeleteSubresources(resource)
    └─ destroy_subresources()
           srv.destroy(), uav.destroy()  ← descriptor 해제
           rtv.destroy(), dsv.destroy()  ← (병합 후) 함께 해제
```

---

## 5.13 정리

| 개념 | 핵심 |
|------|------|
| 파일 디코딩 | PNG/JPG → 선형 픽셀 배열 (CPU 메모리) |
| Mipmap | 해상도별 축소본 세트. LOD 자동 선택. 원본의 1.33배 메모리 |
| 타일드 레이아웃 | GPU DEFAULT heap의 텍스처 메모리 구조. 공간 지역성 최적화 |
| Footprint | Upload heap(선형)에서 텍스처 Row 위치 기술. RowPitch 256 정렬 |
| Subresource | DEFAULT heap 텍스처의 특정 mip/slice/plane 인덱스 |
| BC 압축 | 4×4 픽셀 = 1 블록. 블록 수 = ceiling(pixels / 4) |
| Descriptor | 리소스 해석 방법 (SRV/RTV/DSV/UAV). 같은 리소스에 여러 뷰 가능 |
| Barrier | GPU에 상태 전이 알림. 캐시 flush + 의존성 보장 |
| SwapChain | 화면 출력용 BackBuffer 관리. PRESENT ↔ RENDER_TARGET 전이 |
