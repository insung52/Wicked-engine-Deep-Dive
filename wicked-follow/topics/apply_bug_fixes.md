# VizMotive 적용: Bug Fixes & Stability

## 개요

WickedEngine 분석에서 추출한 7개 버그 수정 커밋을 VizMotive에 적용한 기록.
각 수정은 독립적이지만, **수정 5·6은 texture 주제 적용 이후에 진행 권장** (같은 파일 수정으로 충돌 방지).

### 관련 문서

| 문서 | 내용 |
|------|------|
| [topic_bug_fixes.md](topic_bug_fixes.md) | 원본 WickedEngine 커밋 분석 요약 |
| [apply_texture.md](apply_texture.md) | Resource_DX12 구조 변경 (수정 5·6 선행 조건) |

### 적용 우선순위 및 의존성

| # | 내용 | 의존성 |
|---|------|--------|
| 1 | WaitQueue 제거 (Linux hang) | 없음 |
| 2 | Indirect 버퍼 초기화 (Xbox hang) | 없음 |
| 3 | SWAPCHAIN ResourceState 추가 | 없음 |
| 4 | StringConvert 리팩토링 | 없음 |
| 5 | SRV/UAV 벡터 범위 검사 | **texture 주제 이후 권장** |
| 6 | CopyTexture Undefined Behavior 수정 | **texture 주제 이후 권장** |
| 7 | Clang 경고 수정 | 없음 |

---

## 수정 1: WaitQueue 제거 (Linux hang)

**WickedEngine 커밋**: [dx1 #1 `bf27a9fb`](../dx_1/01_bf27a9fb_linux_hang_fix.md)

### 문제

`WaitQueue()`가 GPU 큐 간 복잡한 의존성을 형성하여 **순환 대기(deadlock)** 발생.
Linux/Vulkan 환경에서 hang으로 나타남.

WickedEngine에서 `WaitQueue`는 **wetmap** 기능(비 내리는 날 표면 젖음 효과)을
별도 COMPUTE 큐에서 처리할 때 사용됐다. 이 구조가 순환 의존성을 만들어 데드락 발생.

### VizMotive 확인

VizMotive에는 **wetmap 기능이 없다.** WickedEngine의 wetmap 제거 맥락은 해당하지 않는다.

VizMotive 내 실제 WaitQueue 사용 현황:

```
Renderer.cpp:385   //device->WaitQueue(cmd, QUEUE_TYPE::QUEUE_COPY);    ← 이미 주석 처리
Renderer.cpp:386   //device->WaitQueue(cmd, QUEUE_TYPE::QUEUE_COMPUTE); ← 이미 주석 처리
Renderer.cpp:1457   device->WaitQueue(cmd, QUEUE_COMPUTE);              ← 아직 활성
```

Line 1457의 용도:
```cpp
device->WaitQueue(cmd, QUEUE_COMPUTE);
// sync to prev frame compute
// (disallow prev frame overlapping a compute task into updating global scene resources for this frame)
```

즉, **이전 프레임의 COMPUTE 작업이 현재 프레임의 씬 리소스 업데이트와 겹치지 않도록** 하는 동기화.

### 왜 이제 제거 가능한가

[apply_frame_fence_sync.md](apply_frame_fence_sync.md)에서 **프레임 경계 큐 간 fence dependency**를 추가했다:

```cpp
// SubmitCommandLists() 끝에서: 모든 큐가 서로의 현재 프레임 완료를 대기
for (int queue1 = 0; queue1 < QUEUE_COUNT; ++queue1)
    for (int queue2 = 0; queue2 < QUEUE_COUNT; ++queue2)
        queues[queue1].queue->Wait(frame_fence[buf][queue2], frame_fence_values[buf]);
```

이 코드가 **"Frame N의 COMPUTE가 완전히 끝나야 Frame N+1의 GRAPHICS가 시작될 수 있다"** 를 GPU 레벨에서 보장한다.

Line 1457의 WaitQueue가 하던 역할과 완전히 동일하므로 **중복**이다.

```
apply_frame_fence_sync 이후:
  Frame N 끝 → 모든 큐 상호 대기 (GPU-GPU fence dependency)
             → Frame N의 COMPUTE 완료 보장
             → Frame N+1의 GRAPHICS 시작

Line 1457 WaitQueue:
  "이전 프레임 COMPUTE 끝날 때까지 대기" ← 위에서 이미 보장됨 → 불필요
```

### 해결

```
제거 대상:
  Renderer.cpp:1457        device->WaitQueue(cmd, QUEUE_COMPUTE) 호출
  GBackendDevice.h:192     virtual void WaitQueue(...) = 0  선언
  GraphicsDevice_DX12.h:368  void WaitQueue(...) override  선언
  GraphicsDevice_DX12.cpp:6095  WaitQueue() 구현
  CommandList_DX12 내      wait_queues 벡터 (존재 시)
```

> **핵심**: 큐 간 동기화는 frame-boundary fence dependency로 충분하다.
> mid-frame WaitQueue는 frame fence sync가 없던 시절의 임시방편이었다.

---

## 수정 2: Indirect 버퍼 초기화 (Xbox hang)

**WickedEngine 커밋**: [dx1 #4 `dc532889`](../dx_1/04_dc532889_xbox_gpu_hang_fix.md)

> 배경 개념: [Indirect Draw란? ExecuteIndirect, 초기화 요구사항](../../study/graphics/part4/part4_6_indirect_draw.md)

### 문제

`ExecuteIndirect()`가 **미초기화된 버퍼의 쓰레기값**을 드로우 파라미터로 해석.
Xbox나 일부 GPU 드라이버에서 "수백만 개를 그려라" 같은 값이 나와 GPU hang 발생.

Windows PC에서는 운 좋게 0으로 초기화된 메모리를 받아 증상이 없었던 것.

### VizMotive 확인

**GraphicsDevice.h 확인 포인트** — `CreateBufferZeroed` 헬퍼 존재 여부:
```cpp
// 있어야 하는 헬퍼
inline bool CreateBufferZeroed(const GPUBufferDesc* desc, GPUBuffer* buffer)
{
    bool result = CreateBuffer(desc, nullptr, buffer);
    // zero-clear copy 수행
    return result;
}
```

**사용처 확인** — Indirect args 버퍼를 생성하는 모든 위치:
```cpp
// 변경 전 (위험)
CreateBuffer(&desc, nullptr, &indirectBuffer);

// 변경 후 (안전)
CreateBufferZeroed(&desc, &indirectBuffer);
```

### 해결

`CreateBufferZeroed()` 헬퍼를 `GraphicsDevice.h`에 추가하고,
모든 `MISC_FLAG::INDIRECT_ARGS` 버퍼 생성 위치에 적용:

```cpp
// GraphicsDevice.h
inline bool CreateBufferZeroed(const GPUBufferDesc* pDesc, GPUBuffer* pBuffer)
{
    bool success = this->CreateBuffer(pDesc, nullptr, pBuffer);
    if (success)
    {
        // 업로드 버퍼를 통해 0으로 초기화
        // (구체적 구현은 WickedEngine 참고)
    }
    return success;
}
```

**적용 위치** (VizMotive 내 `INDIRECT_ARGS` 버퍼 생성 코드 전부):
- 파티클 시스템 indirect 버퍼
- Scene의 SurfelGI, DDGI, impostor 버퍼
- 그 외 GPU-driven rendering 관련 indirect 버퍼

> **교훈**: GPU에서 읽는 버퍼는 **반드시 초기화**해야 한다.
> 특히 indirect 버퍼는 쓰레기값이 GPU hang으로 직결된다.

---

## 수정 3: SWAPCHAIN ResourceState 추가

**WickedEngine 커밋**: [dx1 #6 `4f503da8`](../dx_1/06_4f503da8_hdr_improvements.md)

### 문제

스왑체인 백버퍼 배리어 전환 시 시작/끝 상태(`D3D12_RESOURCE_STATE_PRESENT`)에 대응하는
`ResourceState` enum 값이 없어서, 상태 관리가 모호하고 배리어 전환이 명시적이지 않았다.

### VizMotive 확인

**GBackend.h** `ResourceState` enum 확인:
```cpp
enum class ResourceState : uint32_t
{
    // ... 기존 값들 ...
    // SWAPCHAIN이 없으면 추가 필요
};
```

**GBackendDX12.cpp** — DX12 변환 함수 확인:
```cpp
// _ParseResourceState() 또는 유사 함수에서
// SWAPCHAIN → D3D12_RESOURCE_STATE_PRESENT 매핑이 있는지 확인
```

### 해결

**GBackend.h:**
```cpp
enum class ResourceState : uint32_t
{
    UNDEFINED         = 0,
    // ... 기존 값들 ...
    SWAPCHAIN         = 1 << 17,  // 추가
};
```

**GBackendDX12.cpp** — DX12 변환 함수:
```cpp
case ResourceState::SWAPCHAIN:
    return D3D12_RESOURCE_STATE_PRESENT;
```

**백버퍼 획득 시 layout 설정:**
```cpp
// GetBackBuffer() 또는 동등 함수에서
backbuffer.desc.layout = ResourceState::SWAPCHAIN;
```

> `SWAPCHAIN`이 추가되면 Present → Render Target → Present 배리어 전환이
> 코드상으로 명확해진다.

---

## 수정 4: StringConvert 리팩토링

**WickedEngine 커밋**: [dx1 #13 `e6a003cd`](../dx_1/13_e6a003cd_stringconvert_replacement.md)

### 문제들

기존 `StringConvert()` (wchar_t ↔ char 변환)에 3가지 결함:

| 결함 | 원인 | 위험도 |
|------|------|--------|
| 버퍼 오버플로우 | `dest_size = -1` 기본값 | 높음 |
| Deprecated API | `std::wstring_convert` (C++17 deprecated) | 중간 |
| Dangling Pointer | 임시 객체 `.c_str()` 반환 후 소멸 | 높음 |

```cpp
// 기존 — Dangling Pointer 예시
const wchar_t* result = cv.from_bytes(from).c_str();
//                                    ↑ 임시 객체 — 이 라인 끝에서 소멸!
//                       result는 이미 해제된 메모리를 가리킴
```

### VizMotive 확인

**VizUtils.h / VizHelper.h** 또는 유사한 유틸리티 헤더에서 `StringConvert` 구현 위치 확인:
```cpp
// 검색 키워드: wstring_convert, StringConvert, wchar_t
```

### 해결

커스텀 UTF-8 인코딩/디코딩 구현으로 교체. dest_size를 명시적 파라미터로:

```cpp
// 변경 전 (위험)
std::string  StringConvert(const std::wstring& from);
std::wstring StringConvert(const std::string& from);

// 변경 후 (안전)
void StringConvert(const wchar_t* from, char*    to, size_t to_size);
void StringConvert(const char*    from, wchar_t* to, size_t to_size);

// 사용
char buf[256];
StringConvert(L"파일.txt", buf, sizeof(buf));  // 크기 명시
```

표준 라이브러리 대신 직접 UTF-8 ↔ UTF-16 변환 구현 (~250줄).
코드 분량은 늘지만 deprecated 의존성 제거 + 명시적 크기 관리.

---

## 수정 5: SRV/UAV 벡터 범위 검사

**WickedEngine 커밋**: [dx2 #1 `0bdd7a2a`](../dx_2/01_0bdd7a2a_srv_uav_bounds_check.md)

> **적용 순서**: texture 주제(`apply_texture.md`) 적용 후 진행 권장.
> `GetDescriptorIndex()`와 `Resource_DX12`가 같은 파일에서 수정되기 때문.

### 문제

`GetDescriptorIndex()`가 `subresources_srv[]`, `subresources_uav[]` 벡터에
**범위 검사 없이** 접근 → 잘못된 인덱스 전달 시 out-of-bounds 크래시.

```cpp
// 변경 전 — 위험
case SubresourceType::SRV:
    if (subresource < 0)
        return internal_state->srv.index;
    else
        return internal_state->subresources_srv[subresource].index;  // ⚠️ 범위 검사 없음
```

### VizMotive 확인

**GraphicsDevice_DX12.cpp** 내 `GetDescriptorIndex()` 함수:
```cpp
// 이미 적용됐는지 확인: size() 비교 구문이 있는지 체크
if (subresource >= (int)internal_state->subresources_srv.size())
    return -1;
```

### 해결

**GraphicsDevice_DX12.cpp — `GetDescriptorIndex()`:**
```cpp
int GraphicsDevice_DX12::GetDescriptorIndex(
    const GPUResource* resource, SubresourceType type, int subresource) const
{
    if (resource == nullptr || !resource->IsValid())
        return -1;

    auto internal_state = to_internal(resource);

    switch (type)
    {
    case SubresourceType::SRV:
        if (subresource < 0)
            return internal_state->srv.index;
        else
        {
            if (subresource >= (int)internal_state->subresources_srv.size())
                return -1;  // ✅ 범위 검사
            return internal_state->subresources_srv[subresource].index;
        }

    case SubresourceType::UAV:
        if (subresource < 0)
            return internal_state->uav.index;
        else
        {
            if (subresource >= (int)internal_state->subresources_uav.size())
                return -1;  // ✅ 범위 검사
            return internal_state->subresources_uav[subresource].index;
        }
    }
    return -1;
}
```

**`-1` 반환의 의미**: 유효하지 않은 descriptor — 호출자가 체크해서 스킵 가능.
크래시 대신 안전한 실패(graceful failure).

> **추가 확인**: `RefreshRootDescriptors`, `BindDescriptorTable` 등
> 내부에서 `subresources_srv[]`에 직접 접근하는 위치들도 동일하게 검토 필요.
> (외부 API인 `GetDescriptorIndex()`가 최우선, 내부 코드는 호출자가 제한적)

---

## 수정 6: CopyTexture Undefined Behavior 수정

**WickedEngine 커밋**: [dx2 #6 `ccaaf209`](../dx_2/06_ccaaf209_undefined_behavior_fix.md)

> **적용 순서**: texture 주제(`apply_texture.md`) 적용 후 진행 권장.

### 문제

`CopyTexture(const Texture* dst, ..., const Texture* src, ...)`에서
`Texture*`를 `GPUBuffer*`로 잘못 캐스팅:

```cpp
// 변경 전 — Strict Aliasing Rule 위반 → Undefined Behavior
auto src_internal = to_internal((const GPUBuffer*)src);  // ⚠️ Texture*를 GPUBuffer*로!
auto dst_internal = to_internal((const GPUBuffer*)dst);
```

**왜 기존에 동작했는가**: `internal_state`가 두 구조체 모두 첫 번째 멤버(offset 0)여서
우연히 같은 주소를 가리켰기 때문. 구조체 레이아웃이 바뀌면 즉시 깨짐.

C++ Strict Aliasing Rule: **다른 타입 포인터로 객체에 접근하면 Undefined Behavior**.

### VizMotive 확인

**GraphicsDevice_DX12.cpp** 내 `CopyTexture()` 함수:
```cpp
// to_internal 호출에 잘못된 캐스팅이 있는지 확인
void GraphicsDevice_DX12::CopyTexture(const Texture* dst, ..., const Texture* src, ...)
{
    auto src_internal = to_internal(src);  // ✅ 이렇게 돼 있어야 함
    auto dst_internal = to_internal(dst);
}
```

### 해결

잘못된 캐스팅 제거. VizMotive에는 이미 `to_internal(const Texture*)` 오버로드가 있으므로
캐스팅만 지우면 된다:

```cpp
// 변경 전
auto src_internal = to_internal((const GPUBuffer*)src);

// 변경 후
auto src_internal = to_internal(src);  // Texture* 오버로드 자동 선택
```

**texture 주제 적용 후**: `to_internal(const Texture*)` 반환 타입이
`Texture_DX12*` → `Resource_DX12*`로 바뀌지만, 수정 로직(올바른 오버로드 호출) 자체는 동일.

> **참고**: `CopyBuffer`의 `to_internal((const GPUBuffer*)pSrc)` 캐스팅은 정상.
> 파라미터가 이미 `GPUBuffer*`이므로 UB가 아니다.

---

## 수정 7: Clang 경고 수정

**WickedEngine 커밋**: [dx2 #15 `cf6f3434`](../dx_2/15_cf6f3434_clang_warning_fix.md)

### 문제

`switch`문에 `default` case 누락 → Clang `-Wswitch-default` 경고 발생.

VizMotive는 현재 MSVC(Windows)로만 빌드하지만, MSVC는 기본적으로 이 경고를 내지 않아
잠재적 문제가 숨겨져 있는 상태.

### VizMotive 확인

**GraphicsDevice_DX12.cpp** 내 포맷 변환 switch문:
```cpp
// 경고 발생 패턴: 열거형 전체를 처리하지만 default 없음
switch (format)
{
case DXGI_FORMAT_R8_UNORM:   return Format::R8_UNORM;
case DXGI_FORMAT_R8G8_UNORM: return Format::R8G8_UNORM;
// ... 많은 case들 ...
// default 없음 → Clang이라면 경고
}
```

### 해결

모든 해당 `switch`문에 `default: break;` 추가:

```cpp
switch (format)
{
case DXGI_FORMAT_R8_UNORM:   return Format::R8_UNORM;
// ...
case DXGI_FORMAT_BC7_UNORM_SRGB: return Format::BC7_UNORM_SRGB;
default: break;  // ✅ Clang 경고 해결
}
```

> **왜 중요한가**: 크로스 플랫폼 목표가 아니더라도 경고 0 빌드는 좋은 관행.
> 경고는 잠재적 버그의 지표이며, 나중에 다른 플랫폼으로 이식할 때 비용이 커진다.

---

## 교훈 요약

| # | 수정 | 교훈 |
|---|------|------|
| 1 | WaitQueue 제거 | 큐 간 동기화는 fence로. 별도 Wait API는 설계 냄새(smell) |
| 2 | Indirect 버퍼 초기화 | GPU에서 읽는 버퍼는 반드시 0으로 초기화. `CreateBufferZeroed` 패턴 활용 |
| 3 | SWAPCHAIN 상태 | 모든 리소스 상태를 enum으로 명시적 관리 |
| 4 | StringConvert | 레거시 API(`wstring_convert`) 의존 말고, 크기 명시적 인터페이스로 |
| 5 | 범위 검사 | 벡터/배열 접근 전 항상 `size()` 체크, 실패 시 -1/nullptr 반환 |
| 6 | CopyTexture UB | C-style 캐스팅은 UB의 온상. 올바른 오버로드/타입을 직접 사용 |
| 7 | Clang 경고 | switch default 항상 추가. 경고 0 빌드 유지 |
