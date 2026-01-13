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

## 구현 우선순위

### Week 1: Python 바인딩
- [x] pybind11 설치
- [x] 기본 바인딩 코드 작성 (bind_engine.cpp)
- [ ] VS 2026에서 CMake 프로젝트 빌드
- [ ] Python에서 엔진 초기화 테스트

### Week 2: Python GUI 뷰어
- [ ] DearPyGui 설치
- [ ] 기본 GUI 창 생성
- [ ] VizEngine 렌더링 통합
- [ ] 간단한 씬 조작 (큐브 추가 등)

### Week 3: MCP 서버 통합
- [ ] FastMCP 설치 및 기본 테스트
- [ ] MCP Tools 구현 (create_cube, get_screenshot 등)
- [ ] GUI + MCP 서버 통합
- [ ] 스레드 안정성 확인

### Week 4: Claude 연동 및 테스트
- [ ] Claude Desktop 설정
- [ ] 실제 대화 테스트
- [ ] 버그 수정 및 최적화
- [ ] 문서 작성

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
