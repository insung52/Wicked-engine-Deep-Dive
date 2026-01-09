# VizMotive Engine MCP Integration Plan

## 개요

VizMotive Engine을 MCP (Model Context Protocol) 서버로 통합하여 Claude와 같은 AI 어시스턴트가 3D 렌더링 엔진을 제어할 수 있도록 합니다. 이를 위해 VizMotive C++ API를 Python으로 바인딩하고, MCP 서버를 구현합니다.

## VizMotive Engine 구조 분석

### 주요 컴포넌트

VizMotive는 ECS(Entity Component System) 기반 3D 렌더링 엔진입니다:

1. **VID (Virtual ID)**: `uint64_t` 기반 엔티티 식별자
2. **Scene 컴포넌트**:
   - VzScene: 3D 씬 컨테이너
   - VzActor: 렌더 가능한 액터 (StaticMesh, GSplat, Volume, Sprite, Particle)
   - VzCamera: 카메라 (Perspective, Orthogonal, Intrinsics)
   - VzLight: 조명
3. **Resource 컴포넌트**:
   - VzGeometry: 메시 데이터
   - VzMaterial: 재질
   - VzTexture: 텍스처
   - VzVolume: 볼륨 데이터
4. **Renderer**: 렌더링 파이프라인 관리
5. **Animation**: 애니메이션 시스템

### API 구조

```cpp
// 엔진 초기화
bool InitEngineLib(const ParamMap<std::string>& arguments);
bool DeinitEngineLib();

// 컴포넌트 생성
VzScene* NewScene(const std::string& name);
VzRenderer* NewRenderer(const std::string& name);
VzCamera* NewCamera(const std::string& name, const VID parentVid = 0u);
VzActorStaticMesh* NewActorStaticMesh(const std::string& name, ...);
VzGeometry* NewGeometry(const std::string& name);
VzMaterial* NewMaterial(const std::string& name);

// 렌더링
bool Render(const SceneVID vidScene, const CamVID vidCam, const float dt = -1.f);
```

## MCP (Model Context Protocol) 이해

### MCP란?

MCP는 Anthropic이 개발한 프로토콜로, AI 어시스턴트가 외부 데이터 소스 및 도구와 통신할 수 있도록 하는 표준 인터페이스입니다.

### MCP 서버 구조

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Claude    │ ◄─────► │  MCP Server │ ◄─────► │  VizMotive  │
│  (Client)   │  JSON   │   (Python)  │   FFI   │   Engine    │
└─────────────┘   RPC   └─────────────┘         └─────────────┘
```

### MCP 주요 기능

1. **Resources**: 읽기 가능한 데이터 제공 (예: 씬 정보, 카메라 설정)
2. **Tools**: 실행 가능한 작업 (예: 메시 생성, 렌더링)
3. **Prompts**: 템플릿화된 프롬프트 제공

## 구현 계획

### Phase 1: Python 바인딩 생성

#### 1.1 기술 선택: pybind11

**이유:**
- C++11/14/17 네이티브 지원
- 자동 타입 변환
- STL 컨테이너 지원 (std::vector, std::string, std::map)
- 헤더 온리 라이브러리 (빌드 간편)
- NumPy 지원 (향후 확장성)

**대안:**
- SWIG: 레거시, 복잡한 설정
- Boost.Python: 무거움, 빌드 복잡
- ctypes: 순수 Python, C++ 기능 제한적

#### 1.2 바인딩 구조

```
VizMotive-Engine/
├── EngineCore/
│   └── HighAPIs/
│       ├── VzEngineAPIs.h
│       ├── VzCamera.h
│       ├── VzRenderer.h
│       └── ...
├── PythonBindings/           # 새로 생성
│   ├── CMakeLists.txt
│   ├── src/
│   │   ├── pyvizmotive.cpp  # 메인 바인딩
│   │   ├── bind_engine.cpp  # 엔진 초기화
│   │   ├── bind_scene.cpp   # Scene 관련
│   │   ├── bind_camera.cpp  # Camera 관련
│   │   ├── bind_renderer.cpp # Renderer 관련
│   │   ├── bind_geometry.cpp # Geometry 관련
│   │   └── bind_material.cpp # Material 관련
│   └── tests/
│       └── test_basic.py
└── vz_mcp/                   # 새로 생성
    ├── server.py             # MCP 서버
    ├── tools/                # MCP Tools
    ├── resources/            # MCP Resources
    └── requirements.txt
```

#### 1.3 pybind11 예제

```cpp
// PythonBindings/src/bind_engine.cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include "VzEngineAPIs.h"

namespace py = pybind11;

void bind_engine(py::module& m) {
    m.def("init_engine", &vzm::InitEngineLib,
          py::arg("arguments") = vzm::ParamMap<std::string>(),
          "Initialize the VizMotive engine");

    m.def("deinit_engine", &vzm::DeinitEngineLib,
          "Deinitialize the VizMotive engine");

    m.def("new_scene", &vzm::NewScene,
          py::arg("name"),
          "Create a new scene",
          py::return_value_policy::reference);

    m.def("new_renderer", &vzm::NewRenderer,
          py::arg("name"),
          "Create a new renderer",
          py::return_value_policy::reference);
}
```

```cpp
// PythonBindings/src/bind_camera.cpp
void bind_camera(py::module& m) {
    py::class_<vzm::VzCamera, vzm::VzSceneObject>(m, "Camera")
        .def("set_world_pose", &vzm::VzCamera::SetWorldPose,
             py::arg("pos"), py::arg("view"), py::arg("up"))
        .def("set_perspective_projection",
             &vzm::VzCamera::SetPerspectiveProjection,
             py::arg("zNearP"), py::arg("zFarP"),
             py::arg("fovInDegree"), py::arg("aspectRatio"),
             py::arg("isVertical") = true)
        .def("get_world_pose", [](const vzm::VzCamera& cam) {
            vfloat3 pos, view, up;
            cam.GetWorldPose(pos, view, up);
            return py::make_tuple(
                py::make_tuple(pos.x, pos.y, pos.z),
                py::make_tuple(view.x, view.y, view.z),
                py::make_tuple(up.x, up.y, up.z)
            );
        });
}
```

```python
# Python 사용 예제
import pyvizmotive as vzm

# 엔진 초기화
vzm.init_engine()

# Scene, Renderer, Camera 생성
scene = vzm.new_scene("main_scene")
renderer = vzm.new_renderer("main_renderer")
camera = vzm.new_camera("main_camera")

# 카메라 설정
camera.set_world_pose(
    pos=(0, 0, 5),
    view=(0, 0, -1),
    up=(0, 1, 0)
)
camera.set_perspective_projection(
    zNearP=0.1,
    zFarP=1000.0,
    fovInDegree=60.0,
    aspectRatio=16/9
)

# 렌더링
renderer.render(scene, camera)

# 정리
vzm.deinit_engine()
```

#### 1.4 CMake 설정

```cmake
# PythonBindings/CMakeLists.txt
cmake_minimum_required(VERSION 3.15)
project(pyvizmotive)

find_package(Python REQUIRED COMPONENTS Interpreter Development)
find_package(pybind11 REQUIRED)

pybind11_add_module(pyvizmotive
    src/pyvizmotive.cpp
    src/bind_engine.cpp
    src/bind_scene.cpp
    src/bind_camera.cpp
    src/bind_renderer.cpp
    src/bind_geometry.cpp
    src/bind_material.cpp
)

target_link_libraries(pyvizmotive PRIVATE EngineCore)
target_include_directories(pyvizmotive PRIVATE
    ${CMAKE_SOURCE_DIR}/EngineCore/HighAPIs
)
```

### Phase 2: MCP 서버 구현

#### 2.1 기술 선택

**FastMCP** (권장)
- Python 기반 MCP SDK
- 데코레이터 기반 간단한 API
- 자동 JSON-RPC 핸들링
- 타입 힌팅 지원

```python
from fastmcp import FastMCP
import pyvizmotive as vzm

mcp = FastMCP("VizMotive Engine")

@mcp.tool()
def create_scene(name: str) -> dict:
    """Create a new 3D scene"""
    scene = vzm.new_scene(name)
    return {"scene_id": scene.get_vid(), "name": name}

@mcp.tool()
def create_camera(
    name: str,
    position: list[float],
    look_at: list[float],
    up: list[float] = [0, 1, 0],
    fov: float = 60.0
) -> dict:
    """Create a camera with specified parameters"""
    camera = vzm.new_camera(name)
    view = [
        look_at[0] - position[0],
        look_at[1] - position[1],
        look_at[2] - position[2]
    ]
    camera.set_world_pose(tuple(position), tuple(view), tuple(up))
    camera.set_perspective_projection(0.1, 1000.0, fov, 16/9)
    return {
        "camera_id": camera.get_vid(),
        "name": name,
        "position": position,
        "fov": fov
    }

@mcp.resource("scene://{scene_name}")
def get_scene_info(scene_name: str) -> str:
    """Get scene information"""
    scene = vzm.get_component_by_name(scene_name)
    # Return scene details as JSON string
    pass

if __name__ == "__main__":
    vzm.init_engine()
    mcp.run()
```

#### 2.2 MCP 서버 구조

```
vz_mcp/
├── server.py              # 메인 서버
├── tools/
│   ├── __init__.py
│   ├── scene_tools.py     # Scene 관련 tools
│   ├── camera_tools.py    # Camera 관련 tools
│   ├── renderer_tools.py  # Renderer 관련 tools
│   ├── geometry_tools.py  # Geometry 관련 tools
│   └── material_tools.py  # Material 관련 tools
├── resources/
│   ├── __init__.py
│   └── scene_resources.py # Scene 정보 제공
├── config.json            # MCP 설정
└── requirements.txt
```

#### 2.3 Tool 카테고리

**Scene Management**
- `create_scene`: 씬 생성
- `list_scenes`: 씬 목록 조회
- `delete_scene`: 씬 삭제

**Camera Control**
- `create_camera`: 카메라 생성
- `set_camera_pose`: 카메라 위치/방향 설정
- `set_camera_projection`: 카메라 투영 설정
- `get_camera_info`: 카메라 정보 조회

**Rendering**
- `create_renderer`: 렌더러 생성
- `set_canvas`: 캔버스 크기 설정
- `render_frame`: 프레임 렌더링
- `save_screenshot`: 스크린샷 저장

**Geometry & Materials**
- `create_geometry`: 지오메트리 생성
- `load_model`: 3D 모델 로드
- `create_material`: 재질 생성
- `set_material_properties`: 재질 속성 설정

**Actors**
- `create_static_mesh`: 스태틱 메시 액터 생성
- `create_light`: 조명 생성
- `set_transform`: 변환 행렬 설정

#### 2.4 Claude Desktop 설정

```json
// claude_desktop_config.json
{
  "mcpServers": {
    "vizmotive": {
      "command": "python",
      "args": [
        "C:/graphics/deepdive/Wicked-engine-Deep-Dive/vz_mcp/server.py"
      ],
      "env": {
        "PYTHONPATH": "C:/graphics/vizmotive/my/VizMotive-Engine/build/Release"
      }
    }
  }
}
```

### Phase 3: 통합 테스트

#### 3.1 단위 테스트

```python
# tests/test_bindings.py
import pytest
import pyvizmotive as vzm

def test_engine_init():
    assert vzm.init_engine() == True
    assert vzm.is_valid_engine() == True
    vzm.deinit_engine()

def test_scene_creation():
    vzm.init_engine()
    scene = vzm.new_scene("test_scene")
    assert scene is not None
    assert scene.get_name() == "test_scene"
    vzm.deinit_engine()

def test_camera_creation():
    vzm.init_engine()
    camera = vzm.new_camera("test_camera")
    camera.set_world_pose((0, 0, 5), (0, 0, -1), (0, 1, 0))
    pos, view, up = camera.get_world_pose()
    assert pos == (0, 0, 5)
    vzm.deinit_engine()
```

#### 3.2 MCP 통합 테스트

```python
# tests/test_mcp.py
from mcp import ClientSession
from mcp.client.stdio import stdio_client

async def test_mcp_tools():
    async with stdio_client("python", ["server.py"]) as (read, write):
        async with ClientSession(read, write) as session:
            # Tool 목록 확인
            tools = await session.list_tools()
            assert "create_scene" in [t.name for t in tools]

            # Tool 실행
            result = await session.call_tool("create_scene", {
                "name": "test_scene"
            })
            assert result["scene_id"] > 0
```

#### 3.3 Claude 대화 시나리오

```
User: VizMotive 엔진으로 간단한 3D 씬을 만들어줘.
      카메라는 (0, 2, 5) 위치에서 원점을 바라보게 하고,
      큐브 하나 추가해줘.

Claude: VizMotive 엔진으로 씬을 생성하겠습니다.

[MCP Tool: create_scene(name="demo_scene")]
[MCP Tool: create_camera(name="main_cam", position=[0,2,5],
           look_at=[0,0,0], up=[0,1,0])]
[MCP Tool: create_geometry(name="cube", type="box",
           size=[1,1,1])]
[MCP Tool: create_static_mesh(name="cube_actor",
           geometry="cube")]
[MCP Tool: render_frame(scene="demo_scene",
           camera="main_cam")]

3D 씬을 생성했습니다:
- Scene: demo_scene
- Camera: (0, 2, 5) → (0, 0, 0)
- Cube actor 추가 완료
- 렌더링 완료
```

## 구현 우선순위

### Priority 1 (핵심 기능)
1. ✅ pybind11 설치 및 기본 설정
2. ✅ 엔진 초기화/종료 바인딩
3. ✅ Scene, Camera, Renderer 바인딩
4. ✅ 기본 렌더링 파이프라인
5. ✅ MCP 서버 기본 구조

### Priority 2 (확장 기능)
1. Geometry, Material 바인딩
2. Actor (StaticMesh, Light) 바인딩
3. Transform 조작
4. 모델 로딩 (OBJ, GLTF)
5. MCP Resources 구현

### Priority 3 (고급 기능)
1. Animation 바인딩
2. Particle System
3. Volume Rendering
4. Post-processing 효과
5. 실시간 편집 기능

## 기술적 도전 과제

### 1. 메모리 관리
- **문제**: C++ 객체의 소유권을 Python에서 관리
- **해결**: pybind11의 `return_value_policy` 사용
  - `reference`: C++에서 소유권 유지
  - `take_ownership`: Python이 소유권 획득
  - `automatic`: 자동 결정

### 2. 타입 변환
- **문제**: C++ 구조체 ↔ Python tuple/dict
- **해결**: Lambda wrapper 또는 custom type caster

```cpp
// vfloat3 → Python tuple
py::class_<vfloat3>(m, "Float3")
    .def(py::init<float, float, float>())
    .def_readwrite("x", &vfloat3::x)
    .def_readwrite("y", &vfloat3::y)
    .def_readwrite("z", &vfloat3::z)
    .def("__repr__", [](const vfloat3& v) {
        return "Float3(" + std::to_string(v.x) + ", " +
               std::to_string(v.y) + ", " + std::to_string(v.z) + ")";
    });
```

### 3. 멀티스레딩
- **문제**: Python GIL vs C++ 멀티스레드 렌더링
- **해결**:
  - `py::gil_scoped_release` for long-running C++ calls
  - `py::gil_scoped_acquire` for callbacks

### 4. 에러 핸들링
- **문제**: C++ 예외 → Python 예외
- **해결**: pybind11 자동 변환 + custom exception

```cpp
py::register_exception<vzm::EngineException>(m, "EngineError");

m.def("new_scene", [](const std::string& name) {
    auto scene = vzm::NewScene(name);
    if (!scene) {
        throw vzm::EngineException("Failed to create scene");
    }
    return scene;
});
```

## 필요한 도구 및 라이브러리

### 개발 환경
- Visual Studio 2022 (C++17)
- Python 3.10+
- CMake 3.15+

### Python 라이브러리
```txt
# requirements.txt
pybind11>=2.11.0
fastmcp>=0.1.0
numpy>=1.24.0
pytest>=7.4.0
```

### C++ 라이브러리
- pybind11 (헤더 온리)
- VizMotive EngineCore

## 개발 로드맵

### Week 1-2: Python 바인딩 기초
- [ ] pybind11 설치 및 CMake 설정
- [ ] 엔진 초기화 API 바인딩
- [ ] Scene, Camera, Renderer 기본 바인딩
- [ ] 단위 테스트 작성

### Week 3-4: MCP 서버 구현
- [ ] FastMCP 설정
- [ ] 기본 Tools 구현 (create_scene, create_camera, render)
- [ ] MCP 서버 테스트
- [ ] Claude Desktop 연동

### Week 5-6: 확장 및 최적화
- [ ] Geometry, Material 바인딩
- [ ] Actor 생성 및 조작
- [ ] 모델 로딩 기능
- [ ] 성능 최적화 및 에러 핸들링

### Week 7-8: 문서화 및 배포
- [ ] API 문서 작성
- [ ] 튜토리얼 작성
- [ ] 예제 코드
- [ ] 패키징 (PyPI 배포 고려)

## 참고 자료

### pybind11
- 공식 문서: https://pybind11.readthedocs.io/
- 예제: https://github.com/pybind/python_example

### MCP
- MCP 사양: https://modelcontextprotocol.io/
- FastMCP: https://github.com/jlowin/fastmcp
- MCP Python SDK: https://github.com/anthropics/anthropic-sdk-python

### 유사 프로젝트
- PyOpenGL: OpenGL Python 바인딩
- PyVista: VTK Python 래퍼
- Blender Python API: bpy

## 다음 단계

1. **환경 설정**: pybind11 및 FastMCP 설치
2. **첫 바인딩**: `InitEngineLib`, `NewScene` 바인딩
3. **테스트**: Python에서 엔진 초기화 테스트
4. **MCP 프로토타입**: 간단한 MCP 서버 작성
5. **반복 개발**: 점진적으로 API 확장

---

*작성일: 2026-01-09*
*작성자: Claude Sonnet 4.5 with User*
