# VizMotive Engine MCP Integration Plan (Revised)

## 개요

![alt text](image-1.png)

VizMotive Engine을 **Blender MCP와 동일한 방식**으로 통합합니다. Python GUI 프로그램(뷰어)에서 MCP 서버를 실행하여 Claude가 **실시간으로 3D 씬을 제어하고 확인**할 수 있도록 합니다.

## 핵심 아키텍처 (Blender MCP 방식)

### 블렌더 MCP 구조

```
┌──────────────────────────────────────┐
│   Blender (독립 GUI 프로그램)          │
│                                      │
│   ┌──────────────────────────────┐  │
│   │  Python (Blender 내장)        │  │
│   │  ├─ bpy (Blender API)        │  │
│   │  └─ MCP Server               │◄─┼─── Claude
│   └──────────────────────────────┘  │
│                                      │
│   [3D Viewport - 실시간 렌더링]       │
│   Claude 명령 → 큐브 생성 → 화면에 표시│
│   Screenshot → Claude 확인 → 수정     │
└──────────────────────────────────────┘
```

**핵심**:
- GUI 프로그램 실행 → MCP 서버 시작 → Claude 연동
- Claude 명령 → 실시간 화면 반영 → Claude가 스크린샷으로 확인

### VizMotive MCP 구조 (동일한 방식!)

```
┌──────────────────────────────────────┐
│   VizMotive Viewer (Python GUI)      │
│                                      │
│   ┌──────────────────────────────┐  │
│   │  Python                       │  │
│   │  ├─ pyvizmotive (바인딩)      │  │
│   │  ├─ DearPyGui (GUI)          │  │
│   │  └─ MCP Server               │◄─┼─── Claude
│   └──────────────────────────────┘  │
│          │                           │
│          ▼                           │
│   ┌──────────────────────────────┐  │
│   │  VizEngine.dll (C++)         │  │
│   │  (이미 빌드됨!)               │  │
│   └──────────────────────────────┘  │
│                                      │
│   [3D Viewport - 실시간 렌더링]       │
│   Claude 명령 → 메시 생성 → 화면에 표시│
│   Screenshot → Claude 확인 → 수정     │
└──────────────────────────────────────┘
```

**차이점**:
- 블렌더는 완성된 프로그램, VizMotive는 라이브러리
- **해결책**: Python으로 간단한 뷰어 프로그램 작성 (Sample14의 Python 버전)

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

## 구현 계획 (4 Phase)

### Phase 1: Python 바인딩 생성 (C++ → Python)

#### 1.1 기술 선택: pybind11

**이유:**
- C++11/14/17 네이티브 지원
- 자동 타입 변환
- STL 컨테이너 지원 (std::vector, std::string, std::map)
- 헤더 온리 라이브러리 (빌드 간편)

#### 1.2 디렉토리 구조

```
VizMotive-Engine/
├── EngineCore/
│   └── HighAPIs/
│       ├── VzEngineAPIs.h
│       ├── VzCamera.h
│       ├── VzRenderer.h
│       └── ...
├── Install/                      # 이미 빌드된 엔진
│   ├── bin/
│   │   ├── debug_dll/VizEngined.dll
│   │   └── release_dll/VizEngine.dll
│   ├── lib/
│   │   ├── VizEngined.lib
│   │   └── VizEngine.lib
│   └── vzm2/                    # 헤더 파일
│       └── VzEngineAPIs.h
├── PythonBindings/               # Python 바인딩
│   ├── CMakeLists.txt
│   ├── setup.py
│   ├── src/
│   │   ├── pyvizmotive.cpp      # 메인 바인딩
│   │   ├── bind_engine.cpp      # 엔진 초기화
│   │   ├── bind_scene.cpp       # Scene 관련
│   │   ├── bind_camera.cpp      # Camera 관련
│   │   ├── bind_renderer.cpp    # Renderer 관련
│   │   ├── bind_geometry.cpp    # Geometry 관련
│   │   └── bind_material.cpp    # Material 관련
│   └── tests/
│       └── test_basic.py
└── vz_mcp/                       # MCP 통합 GUI
    ├── vz_viewer_mcp.py          # GUI 뷰어 + MCP 서버
    ├── tools/
    │   ├── __init__.py
    │   ├── scene_tools.py
    │   ├── camera_tools.py
    │   └── render_tools.py
    └── requirements.txt
```

#### 1.3 빌드 방법

**Visual Studio 2026 Insiders에서 CMake 프로젝트 열기**

1. VS 2026 Insiders 실행
2. `File → Open → CMake...`
3. `PythonBindings/CMakeLists.txt` 선택
4. VS가 자동으로 CMake 구성
5. `Build → Build All`
6. 결과: `pyvizmotive.pyd` 생성

#### 1.4 바인딩 예제

```cpp
// PythonBindings/src/bind_engine.cpp
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include "vzm2/VzEngineAPIs.h"

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

    m.def("new_camera", &vzm::NewCamera,
          py::arg("name"),
          py::arg("parent_vid") = 0u,
          "Create a new camera",
          py::return_value_policy::reference);
}
```

```python
# Python에서 사용
import pyvizmotive as vzm

vzm.init_engine()
scene = vzm.new_scene("main")
renderer = vzm.new_renderer("main")
camera = vzm.new_camera("cam")
```

---

### Phase 2: Python GUI 뷰어 작성 (Sample14의 Python 버전)

#### 2.1 기술 선택: DearPyGui

**이유:**
- ImGui 기반 (Sample14와 유사)
- Python 네이티브 라이브러리
- 렌더링 텍스처 표시 가능
- 간단한 API

**대안 고려:**
- Pygame: 너무 기본적, 3D에 부적합
- PyQt5/PySide6: 무겁고 복잡
- Tkinter: 3D 렌더링 통합 어려움

#### 2.2 GUI 뷰어 구조

```python
# vz_mcp/vz_viewer.py (MCP 없는 기본 버전)
import pyvizmotive as vzm
from dearpygui import dearpygui as dpg
import numpy as np

class VizMotiveViewer:
    def __init__(self):
        self.running = False

        # VizMotive 초기화
        vzm.init_engine()

        self.scene = vzm.new_scene("main_scene")
        self.renderer = vzm.new_renderer("main_renderer")
        self.camera = vzm.new_camera("main_camera")

        # 카메라 설정
        self.camera.set_world_pose(
            pos=(0, 2, 5),
            view=(0, 0, -1),
            up=(0, 1, 0)
        )
        self.camera.set_perspective_projection(
            zNearP=0.1,
            zFarP=1000.0,
            fovInDegree=60.0,
            aspectRatio=16/9
        )

        # 렌더러 설정
        self.renderer.set_canvas(1280, 720, 96.0)

    def create_gui(self):
        dpg.create_context()
        dpg.create_viewport(title="VizMotive Viewer", width=1280, height=720)

        with dpg.window(label="VizMotive Viewer", tag="main_window"):
            # 3D 뷰포트
            with dpg.group(horizontal=False):
                dpg.add_text("3D Viewport")
                # 렌더링 결과 표시
                dpg.add_image("render_texture", width=1024, height=576)

            # 컨트롤 패널
            with dpg.collapsing_header(label="Scene Controls"):
                dpg.add_button(label="Add Cube", callback=self.add_cube)
                dpg.add_button(label="Add Light", callback=self.add_light)
                dpg.add_button(label="Clear Scene", callback=self.clear_scene)

        dpg.setup_dearpygui()
        dpg.show_viewport()

    def render_loop(self):
        while dpg.is_dearpygui_running():
            # VizMotive 렌더링
            self.renderer.render(self.scene, self.camera)

            # 렌더링 결과를 텍스처로 가져오기
            # (VizEngine에서 텍스처 데이터 추출 필요)

            # DearPyGui 업데이트
            dpg.render_dearpygui_frame()

    def add_cube(self):
        # 큐브 생성
        cube_geo = vzm.new_geometry("cube")
        cube_mat = vzm.new_material("cube_mat")
        cube = vzm.new_actor_static_mesh("cube_actor", cube_geo, cube_mat)

    def cleanup(self):
        dpg.destroy_context()
        vzm.deinit_engine()

if __name__ == "__main__":
    viewer = VizMotiveViewer()
    viewer.create_gui()
    viewer.render_loop()
    viewer.cleanup()
```

#### 2.3 VizEngine 렌더링 결과 가져오기

**문제**: VizEngine의 렌더링 결과를 DearPyGui에 표시

**해결 방법**:

1. **SharedRenderTarget 사용** (VzRenderer::GetSharedRenderTarget)
   - DirectX12 텍스처 공유
   - DearPyGui에서 DirectX 텍스처 표시

2. **스크린샷 파일 저장** (간단한 방법)
   - VizEngine에서 PNG 저장
   - DearPyGui에서 이미지 로드

```python
# 방법 2: 스크린샷 방식 (간단함)
def render_loop(self):
    while dpg.is_dearpygui_running():
        # VizMotive 렌더링
        self.renderer.render(self.scene, self.camera)

        # 스크린샷 저장
        self.renderer.save_screenshot("temp_render.png")

        # DearPyGui에 표시
        dpg.set_value("render_texture", load_image("temp_render.png"))

        dpg.render_dearpygui_frame()
```

---

### Phase 3: MCP 서버 GUI 통합 (핵심!)

#### 3.1 MCP 서버 통합 구조

```python
# vz_mcp/vz_viewer_mcp.py
import pyvizmotive as vzm
from dearpygui import dearpygui as dpg
from fastmcp import FastMCP
import threading
import asyncio

class VizMotiveMCPViewer:
    def __init__(self):
        # VizMotive 초기화
        vzm.init_engine()
        self.scene = vzm.new_scene("main_scene")
        self.renderer = vzm.new_renderer("main_renderer")
        self.camera = vzm.new_camera("main_camera")

        # MCP 서버
        self.mcp = FastMCP("VizMotive Engine")
        self.setup_mcp_tools()

    def setup_mcp_tools(self):
        """MCP Tools 등록"""

        @self.mcp.tool()
        def create_cube(name: str, position: list[float] = [0, 0, 0]) -> dict:
            """Create a cube in the scene"""
            cube_geo = vzm.new_geometry(f"{name}_geo")
            cube_mat = vzm.new_material(f"{name}_mat")
            cube = vzm.new_actor_static_mesh(name, cube_geo, cube_mat)
            cube.set_position(position)
            return {
                "status": "created",
                "name": name,
                "position": position
            }

        @self.mcp.tool()
        def get_screenshot() -> str:
            """Get current viewport screenshot as base64"""
            self.renderer.save_screenshot("temp_screenshot.png")
            with open("temp_screenshot.png", "rb") as f:
                import base64
                return base64.b64encode(f.read()).decode()

        @self.mcp.tool()
        def set_camera_position(position: list[float], look_at: list[float]) -> dict:
            """Set camera position and target"""
            view = [
                look_at[0] - position[0],
                look_at[1] - position[1],
                look_at[2] - position[2]
            ]
            self.camera.set_world_pose(
                tuple(position),
                tuple(view),
                (0, 1, 0)
            )
            return {"status": "updated"}

    def start_mcp_server(self):
        """MCP 서버를 별도 스레드에서 시작"""
        def run_mcp():
            asyncio.run(self.mcp.run())

        mcp_thread = threading.Thread(target=run_mcp, daemon=True)
        mcp_thread.start()

    def create_gui(self):
        dpg.create_context()
        dpg.create_viewport(title="VizMotive MCP Viewer", width=1280, height=720)

        with dpg.window(label="VizMotive MCP Viewer", tag="main_window"):
            # MCP 상태
            with dpg.group(horizontal=True):
                dpg.add_text("MCP Server Status: ")
                dpg.add_text("Running", tag="mcp_status", color=(0, 255, 0))

            # 3D 뷰포트
            dpg.add_image("render_texture", width=1024, height=576)

            # 씬 정보
            with dpg.collapsing_header(label="Scene Info"):
                dpg.add_text("Objects in scene:", tag="object_count")

        dpg.setup_dearpygui()
        dpg.show_viewport()

    def render_loop(self):
        while dpg.is_dearpygui_running():
            # VizMotive 렌더링
            self.renderer.render(self.scene, self.camera)

            # 화면 업데이트
            dpg.render_dearpygui_frame()

    def run(self):
        """메인 실행 함수"""
        # MCP 서버 시작
        self.start_mcp_server()

        # GUI 생성 및 실행
        self.create_gui()
        self.render_loop()

        # 종료
        dpg.destroy_context()
        vzm.deinit_engine()

if __name__ == "__main__":
    viewer = VizMotiveMCPViewer()
    viewer.run()
```

#### 3.2 사용 시나리오 (블렌더와 동일!)

```
1. 사용자: vz_viewer_mcp.py 실행
   → GUI 창 열림
   → MCP 서버 백그라운드 시작

2. Claude Desktop에서 연결

3. 사용자: "큐브 3개를 다른 위치에 만들어줘"

   Claude:
   [MCP Tool: create_cube(name="cube1", position=[0,0,0])]
   [MCP Tool: create_cube(name="cube2", position=[2,0,0])]
   [MCP Tool: create_cube(name="cube3", position=[-2,0,0])]
   [MCP Tool: get_screenshot()]

   → GUI 창에 실시간으로 큐브 3개 생성됨
   → Claude가 스크린샷 확인

   Claude: "3개의 큐브를 생성했습니다. 각각 x축 방향으로 2씩 떨어져 있습니다."

4. 사용자: "카메라를 더 멀리 옮겨서 전체를 볼 수 있게 해줘"

   Claude:
   [MCP Tool: set_camera_position(position=[0,5,10], look_at=[0,0,0])]
   [MCP Tool: get_screenshot()]

   → GUI 창에서 카메라 위치 실시간 변경
   → Claude가 확인

   Claude: "카메라를 뒤로 이동시켰습니다. 이제 3개의 큐브가 모두 보입니다."
```

---

### Phase 4: Claude Desktop 연동

#### 4.1 Claude Desktop 설정

```json
// C:\Users\<USERNAME>\AppData\Roaming\Claude\claude_desktop_config.json
{
  "mcpServers": {
    "vizmotive": {
      "command": "python",
      "args": [
        "C:/graphics/vizmotive/my/VizMotive-Engine/vz_mcp/vz_viewer_mcp.py"
      ],
      "env": {
        "PYTHONPATH": "C:/graphics/vizmotive/my/VizMotive-Engine/PythonBindings"
      }
    }
  }
}
```

#### 4.2 테스트 시나리오

```python
# tests/test_mcp_integration.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def test_create_cube():
    """Test cube creation through MCP"""
    server_params = StdioServerParameters(
        command="python",
        args=["vz_viewer_mcp.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            # Initialize
            await session.initialize()

            # List available tools
            tools = await session.list_tools()
            assert "create_cube" in [t.name for t in tools.tools]

            # Create a cube
            result = await session.call_tool("create_cube", {
                "name": "test_cube",
                "position": [1, 2, 3]
            })

            assert result.content[0].text == '{"status": "created", ...}'
```

---

## 구현 현황 (2026-01-15 업데이트)

### Phase 1-4: 기본 구현 완료

- [x] pybind11 설치 및 기본 바인딩 코드 작성
- [x] VS에서 CMake 프로젝트 빌드
- [x] Python에서 엔진 초기화 테스트
- [x] DearPyGui 기본 GUI 창 생성
- [x] VizEngine 렌더링 통합 (PNG 파일 방식)
- [x] FastMCP 설치 및 기본 테스트
- [x] MCP Tools 구현 (create_cube, create_sphere, screenshot, clear_scene)
- [x] GUI + MCP 서버 통합 (headless 모드)
- [x] Claude Desktop 설정 및 연동 테스트
- [x] **버퍼 오버플로우 버그 수정** (instanceResLookupUploadBuffer)

### 현재 구현된 Python 바인딩

```python
# 엔진 기본
init_engine(), deinit_engine(), is_valid_engine()
new_scene(), new_renderer(), new_camera(), new_geometry(), new_material()
new_actor_static_mesh(), new_light()
get_first_vid_by_name(), get_component(), get_name_by_vid(), remove_component()

# Geometry 생성
generate_box_geometry(), generate_sphere_geometry(), generate_icosahedron_geometry()

# VzScene
get_vid(), get_name(), append_child()

# VzCamera
set_world_pose(), set_perspective_projection(), set_visible_layer_mask()

# VzRenderer
set_canvas(), resize_canvas(), get_canvas(), set_clear_color()
render(), store_render_target(), store_render_target_to_file()

# VzActor
set_position(), set_scale(), set_visible_layer_mask(), enable_unlit()

# VzMaterial
set_base_color()

# VzLight
set_light_type(), set_position(), set_range(), set_intensity(), set_color()
set_visible_layer_mask()
```

### 현재 구현된 MCP 도구

| 카테고리 | 도구 | 설명 |
|----------|------|------|
| **기본** | `ping()` | 서버 연결 테스트 |
| | `start_viewer()` | 뷰어 GUI 시작 |
| | `get_viewer_status()` | 뷰어 상태 확인 |
| **오브젝트 생성** | `create_cube(x, y, z, size, color)` | 큐브 생성 |
| | `create_sphere(x, y, z, radius, color)` | 구 생성 |
| | `create_light(name, type, pos, color, intensity, range)` | 라이트 생성 |
| **카메라** | `set_camera_position(x, y, z, look_at)` | 카메라 위치/타겟 |
| | `get_camera_info()` | 카메라 정보 |
| **라이트** | `set_light_properties(name, intensity, color, shadow)` | 라이트 속성 |
| **머티리얼** | `set_object_color(name, r, g, b, a)` | 오브젝트 색상 |
| | `set_material_properties(name, wireframe, ...)` | 머티리얼 속성 |
| **변환** | `set_object_position(name, x, y, z)` | 위치 변경 |
| | `set_object_scale(name, sx, sy, sz)` | 스케일 변경 |
| **씬 관리** | `list_objects()` | 오브젝트 목록 |
| | `delete_object(name)` | 오브젝트 삭제 |
| | `clear_scene()` | 씬 초기화 |
| **파일** | `load_model(filepath, name)` | 3D 모델 로드 |
| **렌더링** | `render_screenshot(filename)` | 스크린샷 저장 |
| | `set_render_settings(hdr, tonemap, clear_color)` | 렌더 설정 |

---

## Phase 5: 확장 API 바인딩 (TODO)

### 5.1 Camera 확장 (우선순위: 높음) ✅

- [x] `get_world_pose()` - 현재 카메라 위치/방향 반환
- [x] `get_view_matrix()` - 뷰 행렬 반환
- [x] `get_projection_matrix()` - 프로젝션 행렬 반환
- [x] `set_orthogonal_projection()` - 직교 투영 설정
- [x] `set_intrinsics_projection()` - 내부 파라미터로 투영 설정
- [x] `enable_clipper()` - 클리핑 활성화
- [x] `set_clip_plane()` - 클립 평면 설정
- [x] `set_clip_box()` - 클립 박스 설정
- [x] DVR_TYPE enum 바인딩
- [ ] OrbitalControl 클래스 바인딩 (orbit, pan, zoom)

### 5.2 Renderer 확장 (우선순위: 높음) ✅

- [x] `set_viewport()` - 뷰포트 설정
- [x] `get_viewport()` - 뷰포트 반환
- [x] `set_scissor()` / `get_scissor()` - 시저 영역
- [x] `set_layer_mask()` - 레이어 마스크
- [x] `enable_clear()` - 클리어 활성화
- [x] `skip_postprocess()` - 포스트프로세스 스킵
- [x] `set_allow_hdr()` / `get_allow_hdr()` - HDR 설정
- [x] `set_tonemap()` - 톤맵 설정 (Reinhard, ACES)
- [x] `enable_frame_lock()` - 프레임 잠금
- [x] `picking()` - 오브젝트 피킹
- [x] `unproj_to_world()` - 화면→월드 좌표 변환
- [x] `show_debug_buffer()` - 디버그 버퍼 표시
- [x] `set_render_option_enabled()` - 렌더 옵션 설정
- [x] Tonemap enum 바인딩 (Reinhard, ACES)
- [x] ActorFilter enum 바인딩
- [ ] `set_render_option_value_array()` - 렌더 옵션 값 배열

### 5.3 Material 확장 (우선순위: 중간) ✅

- [x] `set_texture()` - 텍스처 슬롯 설정 (BaseColor, Normal, Emissive 등)
- [x] `set_shader_type()` - 셰이더 타입 (PHONG, PBR, UNLIT, VOLUMEMAP)
- [x] `set_double_sided()` - 양면 렌더링
- [x] `set_shadow_cast()` - 그림자 캐스트
- [x] `set_shadow_receive()` - 그림자 리시브
- [x] `set_wireframe()` - 와이어프레임 모드
- [x] `set_phong_factors()` - Phong 계수
- [x] `set_gaussian_splatting_enabled()` - 가우시안 스플래팅
- [x] `get_base_color()` - 베이스 컬러 getter
- [x] TextureSlot enum 바인딩
- [x] ShaderType enum 바인딩
- [ ] `set_volume_texture()` - 볼륨 텍스처 설정
- [ ] `set_lookup_table()` - 룩업 테이블 설정

### 5.4 Light 확장 (우선순위: 중간) ✅

- [x] `set_spotlight_cone_angle()` - 스포트라이트 콘 각도
- [x] `set_pointlight_length()` - 포인트라이트 길이
- [x] `set_radius()` - 반경
- [x] `set_length()` - 길이
- [x] `enable_visualizer()` - 시각화 도우미
- [x] `enable_cast_shadow()` - 그림자 캐스트 활성화
- [x] `set_cascade_shadow_map_distances()` - 캐스케이드 섀도우맵 거리

### 5.5 VzTexture 클래스 (우선순위: 중간) ✅

- [x] `VzTexture` 클래스 전체 바인딩
- [x] `create_from_image_file()` - 파일에서 텍스처 로드
- [x] `create_from_memory()` - 메모리에서 텍스처 생성
- [x] `create_lookup_texture()` - 룩업 텍스처 생성
- [x] `update_lookup()` - 룩업 업데이트
- [x] `get_texture_size()` - 텍스처 크기
- [x] `set_sampler_filter()` - 샘플러 필터
- [x] TextureFormat enum 바인딩
- [x] SamplerType enum 바인딩
- [ ] `get_texture_format()` - 텍스처 포맷

### 5.6 Actor 확장 (우선순위: 중간)

**VzActor 공통:**
- [ ] `enable_pickable()` - 피킹 활성화
- [ ] `is_pickable()` - 피킹 가능 여부
- [ ] `get_all_materials()` - 모든 머티리얼 반환

**VzActorStaticMesh:**
- [ ] `set_geometry()` - 지오메트리 설정
- [ ] `set_material()` / `set_materials()` - 머티리얼 설정
- [ ] `get_geometry()` / `get_material()` / `get_materials()` - getter
- [ ] `enable_clipper()`, `set_clip_plane()`, `set_clip_box()` - 클리핑
- [ ] `enable_outline()`, `set_outline_thickness()`, `set_outline_color()` - 아웃라인
- [ ] `assign_collider()`, `has_collider()`, `collision_check()` - 충돌 검사
- [ ] `enable_shadows_cast()`, `enable_shadows_receive()` - 그림자

### 5.7 새로운 Actor 타입들 (우선순위: 낮음)

**VzActorVolume (볼륨 렌더링):**
- [ ] 클래스 바인딩
- [ ] `set_material()`, `get_material()`
- [ ] `create_and_set_material()`

**VzActorSprite (2D 스프라이트):**
- [ ] 클래스 바인딩
- [ ] `set_sprite_texture()`
- [ ] `set_sprite_scale()`, `set_sprite_position()`, `set_sprite_rotation()`
- [ ] `set_sprite_opacity()`, `set_sprite_fade()`
- [ ] `enable_camera_facing()`, `enable_camera_scaling()`
- [ ] `enable_depth_test()`

**VzActorSpriteFont (텍스트 렌더링):**
- [ ] 클래스 바인딩
- [ ] `set_text()`
- [ ] `set_font_style()`, `set_font_size()`, `set_font_color()`
- [ ] `set_font_shadow_color()`, `set_font_shadow_offset()`

**VzActorGSplat (가우시안 스플래팅):**
- [ ] 클래스 바인딩
- [ ] `set_geometry()`, `set_material()`
- [ ] `get_geometry()`, `get_material()`

### 5.8 VzAnimation (우선순위: 낮음)

- [ ] 애니메이션 클래스 바인딩
- [ ] 애니메이션 재생/정지/루프 제어

### 5.9 VzArchive - 파일 로딩 (우선순위: 높음) ✅

- [x] `VzArchive` 클래스 바인딩
- [x] `save_file()` - 씬 저장
- [x] `read_file()` - 씬 로드
- [x] `new_archive()` factory 함수
- [x] `load_model_file()` - 모델 로딩 (GLTF/GLB/OBJ 등)
- [x] `new_texture()`, `new_volume()` factory 함수
- [ ] 3D Gaussian Splatting (.ply) 로딩 전용 함수

### 5.10 VzSlicer (의료 영상용, 우선순위: 낮음)

- [ ] 슬라이서 카메라 바인딩
- [ ] 슬라이스 두께 설정
- [ ] 커브드 슬라이서

---

## Phase 6: MCP 도구 확장 (2026-01-15 업데이트)

### 6.1 카메라 제어 도구 ✅
- [x] `set_camera_position(position, look_at)` - 카메라 위치/타겟 설정
- [x] `get_camera_info()` - 카메라 정보 반환
- [ ] `orbit_camera(yaw, pitch)` - 카메라 오비트
- [ ] `pan_camera(dx, dy)` - 카메라 팬
- [ ] `zoom_camera(delta)` - 카메라 줌

### 6.2 오브젝트 변환 도구 ✅
- [x] `set_object_position(name, position)` - 위치 설정
- [x] `set_object_scale(name, scale)` - 스케일 설정
- [ ] `set_object_rotation(name, rotation)` - 회전 설정 (오일러 또는 쿼터니언)
- [ ] `get_object_transform(name)` - 변환 정보 반환

### 6.3 머티리얼 도구 ✅
- [x] `set_object_color(name, color)` - 베이스 컬러 설정
- [x] `set_material_properties(name, wireframe, double_sided, shader_type)` - 머티리얼 속성
- [ ] `set_material_texture(name, slot, texture_path)` - 텍스처 설정

### 6.4 라이트 도구 ✅
- [x] `create_light(name, type, position, color, intensity, range)` - 라이트 생성
- [x] `set_light_properties(name, intensity, color, cast_shadow)` - 라이트 속성

### 6.5 씬 관리 도구 ✅
- [x] `list_objects()` - 씬 오브젝트 목록
- [x] `delete_object(name)` - 오브젝트 삭제
- [x] `clear_scene()` - 씬 초기화
- [ ] `get_object_info(name)` - 오브젝트 정보
- [ ] `duplicate_object(name, new_name)` - 오브젝트 복제

### 6.6 파일 I/O 도구 ✅
- [x] `load_model(filepath, name)` - 3D 모델 임포트
- [ ] `export_scene(filepath)` - 씬 익스포트
- [ ] `load_texture(filepath)` - 텍스처 로드

### 6.7 렌더링 설정 도구 ✅
- [x] `set_render_settings(hdr, tonemap, clear_color)` - 렌더 설정
- [ ] `set_render_resolution(width, height)` - 렌더 해상도
- [ ] `enable_ddgi(enabled)` - DDGI 활성화
- [ ] `set_ddgi_settings(raycount, blend_speed, grid)` - DDGI 설정

### 6.8 피킹/선택 도구
- [ ] `pick_object(x, y)` - 화면 좌표로 오브젝트 선택
- [ ] `select_object(name)` - 이름으로 오브젝트 선택
- [ ] `get_selected_object()` - 선택된 오브젝트 반환

---

## 버그 수정 이력

### 2026-01-14: instanceResLookupUploadBuffer 버퍼 오버플로우 수정
- **파일**: `EngineShaders/ShaderEngine/SceneUpdate_Detail.cpp:1157`
- **원인**: `sizeof(uint)` 대신 `sizeof(ShaderInstanceResLookup)` 사용해야 함
- **증상**: 동적 메시 추가 시 특정 개수에서 D3D12 크래시
- **상세**: [Fix_instanceResLookupUploadBuffer_size.md](Fix_instanceResLookupUploadBuffer_size.md) 참조

---

## 기술적 도전 과제

### 1. VizEngine 렌더링 결과를 Python GUI에 표시

**문제**: C++ DLL의 렌더링 결과를 Python GUI에 실시간 표시

**해결 방법**:
- **Option A**: SharedRenderTarget (DirectX 텍스처 공유) - 복잡하지만 성능 좋음
- **Option B**: 스크린샷 파일 저장/로드 - 간단하지만 느림
- **추천**: Option B로 시작, 나중에 Option A로 최적화

### 2. MCP 서버와 GUI 메인 루프 동시 실행

**문제**: MCP 서버(asyncio)와 DearPyGui(동기 루프)를 같은 프로세스에서 실행

**해결 방법**:
```python
# MCP 서버를 별도 스레드에서 실행
def start_mcp_server():
    def run_mcp():
        asyncio.run(mcp.run())

    thread = threading.Thread(target=run_mcp, daemon=True)
    thread.start()

# GUI 메인 루프는 메인 스레드에서
dpg.start_dearpygui()
```

### 3. 스레드 안전성

**문제**: MCP 서버(백그라운드 스레드)가 VizEngine API 호출 시 충돌 가능

**해결 방법**:
- Python의 threading.Lock 사용
- VizEngine API 호출을 메인 스레드로 위임 (큐 사용)

```python
import queue

command_queue = queue.Queue()

# MCP Tool에서
@mcp.tool()
def create_cube(name, position):
    command_queue.put(("create_cube", name, position))
    return {"status": "queued"}

# GUI 루프에서
while dpg.is_dearpygui_running():
    # 큐에서 명령 처리
    try:
        cmd, *args = command_queue.get_nowait()
        if cmd == "create_cube":
            # 메인 스레드에서 안전하게 실행
            vzm.create_static_mesh(*args)
    except queue.Empty:
        pass

    # 렌더링
    renderer.render(scene, camera)
    dpg.render_dearpygui_frame()
```

---

## 필요한 도구 및 라이브러리

### 개발 환경
- Visual Studio 2026 Insiders (C++17)
- Python 3.13+
- VizMotive Engine (이미 빌드됨)

### Python 라이브러리
```txt
# requirements.txt
pybind11>=3.0.0
dearpygui>=1.11.0
fastmcp>=0.1.0
Pillow>=10.0.0          # 이미지 로드
numpy>=1.24.0
pytest>=7.4.0
pytest-asyncio>=0.21.0
```

### 설치 방법
```bash
pip install -r requirements.txt
```

---

## 다음 단계

1. ✅ **pybind11 설치 완료**
2. ✅ **바인딩 코드 작성 완료**
3. **VS 2026에서 CMake 빌드** ← 여기부터 시작!
4. **Python에서 엔진 초기화 테스트**
5. **DearPyGui 기본 GUI 작성**
6. **MCP 서버 통합**

---

*작성일: 2026-01-12*
*작성자: Claude Sonnet 4.5 with User*
*방식: Blender MCP 동일 구조 (Python GUI + MCP 서버 통합)*
