# Topic: Bug Fixes & Stability (버그 수정)

## 개요

주제별로 분류되지 않는 개별 버그 수정 및 안정성 개선.

## 관련 커밋

| 커밋 | 날짜 | 핵심 변경 |
|------|------|----------|
| [dx1 #1 `bf27a9fb`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/01_bf27a9fb_linux_hang_fix.md) | 2025-03-12 | WaitQueue 제거 (Linux hang) |
| [dx1 #4 `dc532889`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/04_dc532889_xbox_gpu_hang_fix.md) | 2025-03-16 | Indirect 버퍼 초기화 (Xbox hang) |
| [dx1 #6 `4f503da8`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/06_4f503da8_hdr_improvements.md) | 2025-03-21 | SWAPCHAIN ResourceState 추가 |
| [dx1 #13 `e6a003cd`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_1/13_e6a003cd_stringconvert_replacement.md) | 2025-05-20 | StringConvert 리팩토링 |
| [dx2 #1 `0bdd7a2a`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/01_0bdd7a2a_srv_uav_bounds_check.md) | 2025-10-22 | SRV/UAV 벡터 범위 검사 |
| [dx2 #6 `ccaaf209`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/06_ccaaf209_undefined_behavior_fix.md) | 2025-11-17 | CopyTexture UB 수정 |
| [dx2 #15 `cf6f3434`](https://github.com/insung52/Wicked-engine-Deep-Dive/blob/main/wicked-follow/dx_2/15_cf6f3434_clang_warning_fix.md) | 2025-11-26 | Clang 경고 수정 |

---

## 버그 상세

### 1. WaitQueue 제거 - Linux hang (dx1 #1)

**문제**: `WaitQueue()` API가 Linux에서 행(hang) 유발

**원인**: GPU 큐 간 복잡한 의존성 → 데드락

**해결**:
- `WaitQueue()` 함수 완전 제거
- `wait_queues` 벡터 제거
- Wetmap 처리를 동일 커맨드리스트에서 수행

---

### 2. Indirect 버퍼 초기화 - Xbox hang (dx1 #4)

**문제**: Indirect 버퍼 미초기화로 Xbox GPU hang

**원인**: 쓰레기 값 → 수백만 인스턴스 그리려 시도

**해결**:
```cpp
// CreateBufferZeroed() 헬퍼 함수
bool CreateBufferZeroed(GPUBuffer* buffer, const GPUBufferDesc& desc) {
    device->CreateBuffer(&desc, nullptr, buffer);
    // 0으로 초기화하는 복사 수행
}

// 모든 INDIRECT_ARGS 버퍼에 적용
CreateBufferZeroed(&indirectBuffer, desc);
```

---

### 3. SWAPCHAIN ResourceState 추가 (dx1 #6)

**문제**: 스왑체인 배리어 전환 시 시작/끝 상태 불명확

**해결**:
```cpp
// 새 ResourceState 추가
ResourceState::SWAPCHAIN = 1 << 17

// DX12 매핑
case ResourceState::SWAPCHAIN:
    return D3D12_RESOURCE_STATE_PRESENT;

// GetBackBuffer에서 설정
backbuffer.desc.layout = ResourceState::SWAPCHAIN;
```

---

### 4. StringConvert 리팩토링 (dx1 #13)

**문제들**:
1. 버퍼 오버플로우: `dest_size = -1` 기본값
2. Deprecated API: `std::wstring_convert`
3. Dangling Pointer: 임시 객체 `.c_str()`

**해결**: 커스텀 UTF-8 인코딩/디코딩 구현 (~250줄)

```cpp
// 변경 전 (위험)
const wchar_t* result = cv.from_bytes(from).c_str();  // 임시 객체!

// 변경 후 (안전)
void StringConvert(const char* from, wchar_t* to, size_t to_size);
```

---

### 5. SRV/UAV 벡터 범위 검사 (dx2 #1)

**문제**: `GetDescriptorIndex()` 범위 검사 없음 → 크래시

**해결**:
```cpp
// 변경 전
return subresources_srv[subresource].index;

// 변경 후
if (subresource >= subresources_srv.size()) return -1;
return subresources_srv[subresource].index;
```

---

### 6. CopyTexture Undefined Behavior (dx2 #6)

**문제**: 잘못된 타입 캐스팅 (Strict Aliasing Rule 위반)

```cpp
// 변경 전 (UB)
auto* src_internal = to_internal((GPUBuffer*)src);  // Texture*를 GPUBuffer*로!

// 변경 후 (안전)
auto* src_internal = to_internal(src);  // 올바른 오버로드 호출
```

---

### 7. Clang 경고 수정 (dx2 #15)

**문제**: switch문 default case 누락

```cpp
// 변경 전
switch (format) {
    case A: return X;
    case B: return Y;
    // default 없음 → Clang 경고
}

// 변경 후
switch (format) {
    case A: return X;
    case B: return Y;
    default: break;
}
```

---

## 교훈

### 1. 초기화 필수
GPU에서 읽는 버퍼는 반드시 초기화.
특히 Indirect 버퍼는 쓰레기 값으로 GPU hang 유발 가능.

### 2. 크로스 플랫폼 테스트
Linux, Xbox 등 다른 플랫폼에서 다르게 동작 가능.
Windows에서 우연히 작동해도 UB일 수 있음.

### 3. 방어적 프로그래밍
- 배열 접근 전 범위 검사
- nullptr 체크
- 올바른 타입 캐스팅

### 4. 컴파일러 경고 0
모든 컴파일러(MSVC, GCC, Clang)에서 경고 없이 빌드되어야 함.
경고는 잠재적 버그 지표.

---

## VizMotive 적용

모든 커밋 적용 완료.
