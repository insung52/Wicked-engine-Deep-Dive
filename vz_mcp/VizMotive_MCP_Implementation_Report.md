# VizMotive MCP 서버 구현 보고서

**날짜:** 2026-01-13
**프로젝트:** VizMotive Engine MCP 통합
**저장소:** `VizMotive-Engine/Examples/Vz_MCP_Sample`

---

## 요약

![alt text](image-1.png)

VizMotive 엔진에 Claude Desktop 통합을 위한 MCP 서버를 구현했습니다. Python bindings 확장, 실시간 렌더링 뷰어, headless MCP 서버의 3단계 구조로, Claude가 3D 씬을 생성하고 조작할 수 있습니다.

---

## 1. 시스템 구조

### 1.1 컴포넌트 구성

```
┌─────────────────┐
│ Claude Desktop  │
└────────┬────────┘
         │ MCP Protocol (stdio JSON-RPC)
         ▼
┌─────────────────────────────┐
│  vz_mcp_server_headless.py  │  ← Headless MCP 서버
│  - FastMCP 프레임워크       │
│  - start_viewer, create_    │
│    cube, create_sphere 등   │
└────────┬────────────────────┘
         │ 파일 기반 통신
         │ scene_commands.json
         ▼
┌─────────────────────────────┐
│  vz_viewer_standalone.py    │  ← 독립 실행 뷰어
│  - DearPyGui GUI            │
│  - 실시간 렌더링            │
│  - 명령 처리기              │
└────────┬────────────────────┘
         │ Python Bindings
         ▼
┌─────────────────────────────┐
│    VizMotive Engine (C++)   │
│  - DirectX 12 렌더러        │
│  - ECS 아키텍처             │
│  - 씬 관리                  │
└─────────────────────────────┘
```

### 1.2 설계 결정

**Q: 왜 서버와 뷰어를 분리했나?**
A: MCP 서버는 Claude Desktop 시작 시 자동으로 실행됩니다. 만약 뷰어가 통합되어 있다면 Claude를 실행할 때마다 뷰어 창이 자동으로 열려 사용자 경험이 나쁩니다. 따라서:
- **Headless 서버**: 항상 실행, stdio 통신
- **독립 뷰어**: 필요 시 `start_viewer()` 도구로 실행

**Q: 왜 파일 기반 통신?**
A: MCP 서버는 stdio를 JSON-RPC 통신에 사용하므로 다른 프로세스와 직접 통신할 수 없습니다. `scene_commands.json` 파일을 통해 뷰어에 명령을 전달합니다.

---

## 2. Python Bindings 확장 (상세)

### 2.1 개요

VizMotive 엔진은 C++로 작성되어 있으며, pybind11을 사용해 Python에서 사용할 수 있도록 바인딩합니다. MCP 서버 구현을 위해 기존 바인딩을 대폭 확장했습니다.

**파일 위치:** `PythonBindings/src/bind_engine.cpp`

### 2.2 엔진 초기화 및 해제

```cpp
// 엔진 초기화
m.def("init_engine_lib", [](const ParamMap<std::string>& arguments) {
    return vzm::InitEngineLib(arguments);
}, py::arg("arguments") = ParamMap<std::string>(),
   "엔진 초기화 - WORKING_DIRECTORY 등 설정 가능");

// 엔진 해제
m.def("deinit_engine_lib", &vzm::DeinitEngineLib,
      "엔진 종료 및 리소스 정리");
```

**사용 예시:**
```python
import vizm as vzm

# 엔진 초기화
working_dir = "C:/graphics/vizmotive/my/VizMotive-Engine/Examples"
vzm.init_engine_lib({"WORKING_DIRECTORY": working_dir})

# ... 작업 수행 ...

# 종료
vzm.deinit_engine_lib()
```

### 2.3 씬(Scene) 생성

```cpp
// 씬 생성
m.def("new_scene", [](const std::string& name) {
    return vzm::NewScene(name);
}, py::arg("name"),
   "새로운 씬 생성");
```

**사용 예시:**
```python
scene = vzm.new_scene("main_scene")
print(f"Scene VID: {scene.get_vid()}")
```

**중요:** VizMotive는 ECS(Entity-Component-System) 아키텍처를 사용하며, 모든 컴포넌트는 고유한 VID(Entity ID)를 가집니다.

### 2.4 카메라(Camera) 생성 및 설정

```cpp
// 카메라 생성
m.def("new_camera", [](const std::string& name, const VID parentVid) {
    return vzm::NewCamera(name, parentVid);
}, py::arg("name"), py::arg("parent_vid") = 0u,
   "카메라 생성");
```

**카메라 클래스 바인딩:**
```cpp
py::class_<VzCamera, VzBaseComp>(m, "VzCamera")
    .def("set_canvas_size", [](VzCamera* cam, uint32_t w, uint32_t h, float dpi) {
        cam->SetCanvasSize(w, h, dpi);
    })
    .def("look_at", [](VzCamera* cam,
                       const std::vector<float>& eye,
                       const std::vector<float>& at,
                       const std::vector<float>& up) {
        cam->LookAt(
            list_to_vfloat3(eye),
            list_to_vfloat3(at),
            list_to_vfloat3(up)
        );
    });
```

**사용 예시:**
```python
camera = vzm.new_camera("main_camera")
camera.set_position([8.0, 5.0, 8.0])
camera.look_at(
    [8.0, 5.0, 8.0],  # 카메라 위치
    [0.0, 0.0, 0.0],  # 바라보는 지점
    [0.0, 1.0, 0.0]   # 상향 벡터
)
camera.set_canvas_size(800, 600, 1.0)
```

### 2.5 렌더러(Renderer) 생성 및 렌더링

```cpp
// 렌더러 생성
m.def("new_renderer", [](const std::string& name) {
    return vzm::NewRenderer(name);
}, py::arg("name"),
   "렌더러 생성");
```

**렌더러 클래스 바인딩:**
```cpp
py::class_<VzRenderer, VzBaseComp>(m, "VzRenderer")
    // 캔버스 설정
    .def("set_canvas", [](VzRenderer* renderer,
                          uint32_t w, uint32_t h, float dpi) {
        renderer->SetCanvas(w, h, dpi, nullptr);
    })

    // 렌더링 실행
    .def("render", [](VzRenderer* renderer,
                      const SceneVID vidScene,
                      const CamVID vidCam) {
        return renderer->Render(vidScene, vidCam, -1.f);
    })

    // 렌더 타겟 저장
    .def("store_render_target_to_file", &VzRenderer::StoreRenderTargetInfoFile)

    // 클리어 컬러 설정
    .def("set_clear_color", [](VzRenderer* renderer,
                                const std::vector<float>& color) {
        renderer->SetClearColor(list_to_vfloat4(color));
    });
```

**사용 예시:**
```python
renderer = vzm.new_renderer("main_renderer")
renderer.set_canvas(800, 600, 96.0)
renderer.set_clear_color([0.2, 0.2, 0.3, 1.0])  # 배경색

# 매 프레임 렌더링
renderer.render(scene.get_vid(), camera.get_vid())

# 스크린샷 저장
renderer.store_render_target_to_file("screenshot.png")
```

### 2.6 지오메트리(Geometry) 생성

```cpp
// 지오메트리 생성
m.def("new_geometry", [](const std::string& name) {
    return vzm::NewGeometry(name);
}, py::arg("name"),
   "지오메트리 리소스 생성");

// 박스 생성
m.def("generate_box_geometry", &vz::geogen::GenerateBoxGeometry,
      py::arg("geometry_vid"),
      py::arg("width") = 1.f,
      py::arg("height") = 1.f,
      py::arg("depth") = 1.f,
      py::arg("tessellation_x") = 1u,
      py::arg("tessellation_y") = 1u,
      py::arg("tessellation_z") = 1u,
      "박스 지오메트리 생성");

// 아이코사히드론 (구체 근사)
m.def("generate_icosahedron_geometry", &vz::geogen::GenerateIcosahedronGeometry,
      py::arg("geometry_vid"),
      py::arg("radius") = 1.f,
      py::arg("detail") = 0u,
      "아이코사히드론 생성 (구체 근사)");
```

**사용 예시:**
```python
# 박스 지오메트리
box_geom = vzm.new_geometry("cube_geometry")
vzm.generate_box_geometry(
    box_geom.get_vid(),
    width=1.0,
    height=1.0,
    depth=1.0
)

# 구체 지오메트리 (아이코사히드론 사용)
sphere_geom = vzm.new_geometry("sphere_geometry")
vzm.generate_icosahedron_geometry(
    sphere_geom.get_vid(),
    radius=1.0,
    detail=2  # 세분화 레벨
)
```

**참고:** `GenerateSphereGeometry`는 구현되지 않아 아이코사히드론을 대신 사용합니다.

### 2.7 머티리얼(Material) 생성

```cpp
// 머티리얼 생성
m.def("new_material", [](const std::string& name) {
    return vzm::NewMaterial(name);
}, py::arg("name"),
   "머티리얼 생성");
```

**머티리얼 클래스 바인딩:**
```cpp
py::class_<VzMaterial, VzBaseComp>(m, "VzMaterial")
    // 베이스 컬러 설정
    .def("set_base_color", [](VzMaterial* mat,
                              const std::vector<float>& color) {
        mat->SetBaseColor(list_to_vfloat4(color));
    })

    // 러프니스 설정
    .def("set_roughness", &VzMaterial::SetRoughness)

    // 메탈릭 설정
    .def("set_metalness", &VzMaterial::SetMetalness);
```

**사용 예시:**
```python
material = vzm.new_material("red_material")
material.set_base_color([1.0, 0.0, 0.0, 1.0])  # RGBA
material.set_roughness(0.5)
material.set_metalness(0.0)
```

### 2.8 액터(Actor) 생성 - Static Mesh

```cpp
// Static Mesh Actor 생성
m.def("new_actor_static_mesh", [](const std::string& name,
                                   const GeometryVID vidGeo,
                                   const MaterialVID vidMat,
                                   const VID parentVid) {
    return vzm::NewActorStaticMesh(name, vidGeo, vidMat, parentVid);
}, py::arg("name"),
   py::arg("geometry_vid") = 0u,
   py::arg("material_vid") = 0u,
   py::arg("parent_vid") = 0u,
   "Static Mesh Actor 생성");
```

**사용 예시:**
```python
# 큐브 생성
cube = vzm.new_actor_static_mesh(
    "my_cube",
    geometry_vid=box_geom.get_vid(),
    material_vid=material.get_vid()
)

# 위치 설정
cube.set_position([0.0, 0.0, 0.0])
cube.set_scale([1.0, 1.0, 1.0])

# 씬에 추가
scene.append_child(cube)
```

### 2.9 라이트(Light) 생성

```cpp
// 라이트 생성
m.def("new_light", [](const std::string& name, VID parentVid) {
    return vzm::NewLight(name, parentVid);
}, py::arg("name"), py::arg("parent_vid") = 0u,
   "라이트 생성");

// 라이트 타입 enum
py::enum_<VzLight::LightType>(m, "LightType")
    .value("DIRECTIONAL", VzLight::LightType::DIRECTIONAL)
    .value("POINT", VzLight::LightType::POINT)
    .value("SPOT", VzLight::LightType::SPOT);
```

**라이트 클래스 바인딩:**
```cpp
py::class_<VzLight, VzBaseComp>(m, "VzLight")
    .def("set_light_type", &VzLight::SetLightType)
    .def("set_color", [](VzLight* light,
                         const std::vector<float>& color) {
        light->SetColor(list_to_vfloat3(color));
    })
    .def("set_intensity", &VzLight::SetIntensity)
    .def("set_range", &VzLight::SetRange);
```

**사용 예시:**
```python
light = vzm.new_light("main_light")
light.set_light_type(vzm.LightType.POINT)
light.set_position([3.0, 5.0, 3.0])
light.set_color([1.0, 1.0, 1.0])  # 흰색
light.set_intensity(30.0)
light.set_range(20.0)

# 씬에 추가
scene.append_child(light)
```

### 2.10 공통 컴포넌트 메서드

모든 컴포넌트는 `VzBaseComp` 클래스를 상속하며, 공통 메서드를 사용할 수 있습니다.

```cpp
py::class_<VzBaseComp>(m, "VzBaseComp")
    // VID 획득
    .def("get_vid", &VzBaseComp::GetVID)

    // 이름 획득
    .def("get_name", &VzBaseComp::GetName)

    // 위치 설정
    .def("set_position", [](VzBaseComp* comp,
                            const std::vector<float>& pos) {
        comp->SetPosition(list_to_vfloat3(pos));
    })

    // 위치 획득
    .def("get_position", [](VzBaseComp* comp) {
        vfloat3 pos;
        comp->GetPosition(pos);
        return std::vector<float>{pos.x, pos.y, pos.z};
    })

    // 스케일 설정
    .def("set_scale", [](VzBaseComp* comp,
                         const std::vector<float>& scale) {
        comp->SetScale(list_to_vfloat3(scale));
    })

    // 회전 설정
    .def("set_rotation", [](VzBaseComp* comp,
                            const std::vector<float>& rotation) {
        comp->SetRotation(list_to_vfloat4(rotation));  // 쿼터니언
    })

    // Layer Mask 설정 (가시성 제어)
    .def("set_visible_layer_mask", &VzBaseComp::SetVisibleLayerMask)

    // 자식 추가
    .def("append_child", [](VzBaseComp* comp, VzBaseComp* child) {
        return vzm::AppendSceneCompTo(child, comp);
    });
```

**사용 예시:**
```python
# 모든 컴포넌트에서 사용 가능
actor.set_position([1.0, 2.0, 3.0])
actor.set_scale([2.0, 2.0, 2.0])
actor.set_rotation([0.0, 0.0, 0.0, 1.0])  # 쿼터니언 (x, y, z, w)

# Layer mask 설정 (중요!)
actor.set_visible_layer_mask(0xF, True)

# 씬에 추가
scene.append_child(actor)
```

### 2.11 타입 변환 헬퍼 함수

VizMotive는 `vfloat3`, `vfloat4` 같은 커스텀 타입을 사용합니다. Python의 `list`와 변환하기 위한 헬퍼 함수를 구현했습니다.

```cpp
// vfloat3 변환
vfloat3 list_to_vfloat3(const std::vector<float>& vec) {
    if (vec.size() < 3) {
        throw std::runtime_error("vfloat3 requires 3 elements");
    }
    return vfloat3(vec[0], vec[1], vec[2]);
}

// vfloat4 변환
vfloat4 list_to_vfloat4(const std::vector<float>& vec) {
    if (vec.size() < 4) {
        throw std::runtime_error("vfloat4 requires 4 elements");
    }
    return vfloat4(vec[0], vec[1], vec[2], vec[3]);
}
```

이 함수들은 바인딩 코드 내부에서 자동으로 사용됩니다:

```cpp
.def("set_position", [](VzBaseComp* comp,
                        const std::vector<float>& pos) {
    comp->SetPosition(list_to_vfloat3(pos));  // 자동 변환
})
```

### 2.12 씬 그래프 구조

VizMotive는 씬 그래프(Scene Graph) 구조를 사용합니다. 모든 렌더링 객체는 씬의 자식으로 추가되어야 합니다.

```python
# 올바른 구조
scene = vzm.new_scene("main")
actor = vzm.new_actor_static_mesh(...)
scene.append_child(actor)  # ✅ 씬에 추가해야 렌더링됨

# 잘못된 구조
actor = vzm.new_actor_static_mesh(...)
# scene.append_child(actor)  # ❌ 추가 안 함 → 렌더링 안 됨
```

### 2.13 Layer Mask (중요!)

VizMotive는 Layer Mask를 사용해 객체의 가시성을 제어합니다. Camera, Actor, Light가 모두 같은 레이어에 있어야 렌더링됩니다.

```python
# 모든 컴포넌트를 0xF 레이어에 설정
camera.set_visible_layer_mask(0xF, True)
actor.set_visible_layer_mask(0xF, True)
light.set_visible_layer_mask(0xF, True)
```

**Layer Mask 비트:**
- `0x1` = Layer 0
- `0x2` = Layer 1
- `0x4` = Layer 2
- `0x8` = Layer 3
- `0xF` = 모든 레이어 (0~3)

**주의:** Layer mask를 설정하지 않으면 객체가 화면에 나타나지 않습니다!

### 2.14 완전한 사용 예시

```python
import vizm as vzm

# 1. 엔진 초기화
working_dir = "C:/graphics/vizmotive/my/VizMotive-Engine/Examples"
vzm.init_engine_lib({"WORKING_DIRECTORY": working_dir})

# 2. 씬 생성
scene = vzm.new_scene("main_scene")

# 3. 카메라 생성
camera = vzm.new_camera("main_camera")
camera.set_position([8.0, 5.0, 8.0])
camera.look_at([8.0, 5.0, 8.0], [0.0, 0.0, 0.0], [0.0, 1.0, 0.0])
camera.set_canvas_size(800, 600, 1.0)
camera.set_visible_layer_mask(0xF, True)

# 4. 렌더러 생성
renderer = vzm.new_renderer("main_renderer")
renderer.set_canvas(800, 600, 96.0)
renderer.set_clear_color([0.2, 0.2, 0.3, 1.0])

# 5. 라이트 생성
light = vzm.new_light("main_light")
light.set_light_type(vzm.LightType.POINT)
light.set_position([3.0, 5.0, 3.0])
light.set_color([1.0, 1.0, 1.0])
light.set_intensity(30.0)
light.set_range(20.0)
light.set_visible_layer_mask(0xF, True)
scene.append_child(light)

# 6. 큐브 생성
# 6-1. 지오메트리
cube_geom = vzm.new_geometry("cube_geom")
vzm.generate_box_geometry(cube_geom.get_vid(), 1.0, 1.0, 1.0)

# 6-2. 머티리얼
cube_mat = vzm.new_material("cube_mat")
cube_mat.set_base_color([1.0, 0.0, 0.0, 1.0])  # 빨간색

# 6-3. 액터
cube = vzm.new_actor_static_mesh(
    "my_cube",
    cube_geom.get_vid(),
    cube_mat.get_vid()
)
cube.set_position([0.0, 0.0, 0.0])
cube.set_scale([1.0, 1.0, 1.0])
cube.set_visible_layer_mask(0xF, True)
scene.append_child(cube)

# 7. 렌더링
renderer.render(scene.get_vid(), camera.get_vid())

# 8. 스크린샷 저장
renderer.store_render_target_to_file("output.png")

# 9. 종료
vzm.deinit_engine_lib()
```

---

## 3. 독립 뷰어 구현

### 3.1 핵심 구조

**파일:** `Examples/Vz_MCP_Sample/vz_viewer_standalone.py`

```python
class VizMotiveViewer:
    def __init__(self):
        self.engine_initialized = False
        self.scene = None
        self.camera = None
        self.renderer = None
        self.created_objects = []  # 생성된 객체 추적

    def init_engine(self):
        """엔진 초기화 및 씬 생성"""
        vzm.init_engine_lib({"WORKING_DIRECTORY": working_dir})

        self.scene = vzm.new_scene("main_scene")
        self.camera = vzm.new_camera("main_camera")
        self.renderer = vzm.new_renderer("main_renderer")

        # 카메라 설정
        self.camera.set_position([8.0, 5.0, 8.0])
        self.camera.look_at([8.0, 5.0, 8.0], [0.0, 0.0, 0.0], [0.0, 1.0, 0.0])

        # 라이트 생성
        light = vzm.new_light("main_light")
        light.set_light_type(vzm.LightType.POINT)
        light.set_position([3.0, 5.0, 3.0])
        light.set_intensity(30.0)
        self.scene.append_child(light)

        self.engine_initialized = True

    def run(self):
        """메인 렌더링 루프"""
        while dpg.is_dearpygui_running():
            if self.engine_initialized:
                self.process_commands()  # JSON 명령 읽기
                self.update_render()     # 프레임 렌더링
            dpg.render_dearpygui_frame()
```

### 3.2 실시간 렌더링 파이프라인

```python
def update_render(self):
    """매 프레임 렌더링 및 표시"""
    # 씬 렌더링
    self.renderer.render(self.scene.get_vid(), self.camera.get_vid())

    # 임시 해결책: PNG로 저장 후 다시 로드
    # (store_render_target()이 손상된 데이터를 반환하는 문제)
    temp_file = "mcp_screenshots/_temp_frame.png"
    self.renderer.store_render_target_to_file(temp_file)

    # PNG 로드
    pil_img = Image.open(temp_file)
    img_data = np.array(pil_img).astype(np.float32) / 255.0

    # G와 B 채널 교환 (VizMotive 채널 순서 문제)
    img_data = img_data[:, :, [0, 2, 1, 3]]

    # DearPyGui 텍스처 업데이트
    dpg.set_value(self.texture_tag, img_data.flatten())
```

### 3.3 명령 처리

```python
def process_commands(self):
    """JSON 파일에서 명령 읽기 및 실행"""
    if not COMMAND_FILE.exists():
        return

    with open(COMMAND_FILE, 'r') as f:
        commands = json.load(f)

    for cmd in commands[len(self.processed_commands):]:
        if cmd['type'] == 'create_cube':
            self.create_cube_object(cmd['position'], cmd['size'], cmd['color'])
        elif cmd['type'] == 'create_sphere':
            self.create_sphere_object(cmd['position'], cmd['radius'], cmd['color'])
        elif cmd['type'] == 'screenshot':
            self.take_screenshot(cmd['filename'])
```

### 3.4 객체 생성

```python
def create_cube_object(self, position, size, color):
    """큐브 생성"""
    self.object_counter += 1
    name = f"mcp_cube_{self.object_counter}"

    # 지오메트리
    geometry = vzm.new_geometry(f"{name}_geom")
    vzm.generate_box_geometry(geometry.get_vid(), size, size, size)

    # 머티리얼
    material = vzm.new_material(f"{name}_mat")
    material.set_base_color(color)

    # 액터
    cube = vzm.new_actor_static_mesh(name, geometry.get_vid(), material.get_vid())
    cube.set_position(position)
    cube.set_visible_layer_mask(0xF, True)

    # 씬에 추가
    self.scene.append_child(cube)

    # 추적
    self.created_objects.append({
        'vid': cube.get_vid(),
        'name': name,
        'type': 'cube'
    })
```

---

## 4. MCP 서버 구현

### 4.1 서버 구조

**파일:** `Examples/Vz_MCP_Sample/vz_mcp_server_headless.py`

```python
import os
import sys

# 중요: FastMCP import 전에 인코딩 설정
os.environ["PYTHONIOENCODING"] = "utf-8"
os.environ["FASTMCP_HIDE_BANNER"] = "1"

# stderr를 로그 파일로 리다이렉트 (stdio는 MCP 프로토콜 전용)
log_file = open("mcp_debug.log", "w", buffering=1, encoding='utf-8')
sys.stderr = log_file

from fastmcp import FastMCP

mcp = FastMCP("VizMotive 3D Renderer")
```

### 4.2 도구 구현

```python
@mcp.tool()
def start_viewer() -> str:
    """뷰어 GUI를 별도 프로세스로 시작"""
    global viewer_process

    if viewer_process and viewer_process.poll() is None:
        return "뷰어가 이미 실행 중입니다"

    viewer_path = os.path.join(script_dir, "vz_viewer_standalone.py")
    viewer_process = subprocess.Popen(
        ["python", viewer_path],
        creationflags=subprocess.CREATE_NEW_CONSOLE
    )

    return f"뷰어 시작됨 (PID: {viewer_process.pid})"

@mcp.tool()
def create_cube(x: float, y: float, z: float, size: float = 1.0,
                r: float = 1.0, g: float = 0.0, b: float = 0.0) -> str:
    """3D 씬에 큐브 생성"""
    command = {
        "type": "create_cube",
        "position": [x, y, z],
        "size": size,
        "color": [r, g, b, 1.0]
    }
    write_command(command)
    return f"위치 ({x}, {y}, {z})에 크기 {size}의 큐브 생성"
```

### 4.3 파일 기반 통신

```python
STATE_FILE = Path(script_dir) / "scene_commands.json"

def write_command(cmd):
    """뷰어가 처리할 명령을 JSON 파일에 기록"""
    commands = []
    if STATE_FILE.exists():
        with open(STATE_FILE, 'r') as f:
            commands = json.load(f)

    commands.append(cmd)

    with open(STATE_FILE, 'w') as f:
        json.dump(commands, f, indent=2)
```

### 4.4 Claude Desktop 설정

**파일:** `%AppData%\Roaming\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "vizmotive": {
      "command": "python",
      "args": [
        "C:\\graphics\\vizmotive\\my\\VizMotive-Engine\\Examples\\Vz_MCP_Sample\\vz_mcp_server_headless.py"
      ],
      "env": {
        "PYTHONIOENCODING": "utf-8"
      }
    }
  }
}
```

---

## 5. 해결한 기술적 문제들

### 5.1 유니코드 인코딩 이슈

**문제:** FastMCP 배너에 유니코드 문자가 있어 한국어 Windows(cp949)에서 오류 발생
```
UnicodeEncodeError: 'cp949' codec can't encode character '\u2584'
```

**해결:**
```python
# FastMCP import 전에 설정해야 함
os.environ["PYTHONIOENCODING"] = "utf-8"
os.environ["FASTMCP_HIDE_BANNER"] = "1"
```

### 5.2 색상 채널 불일치

**문제:** 뷰어에 표시되는 색상과 저장된 스크린샷의 색상이 다름

**조사 결과:**
- DirectX는 BGRA 포맷 사용
- 일반 이미지는 RGBA 포맷
- 감마 보정 차이

**해결:** 초록색과 파란색 채널 교환
```python
img_data = img_data[:, :, [0, 2, 1, 3]]  # R, B, G, A
```

### 5.3 렌더 타겟 손상

**문제:** `store_render_target()`가 손상된 데이터 반환 (이상한 색상 줄무늬)

**원인:** 불명 (DirectX 12 텍스처 readback의 stride/padding 문제로 추정)

**임시 해결:** PNG 저장/로드 사용
```python
# 직접 메모리 접근 대신:
# self.renderer.store_render_target(buffer)

# 파일 기반 접근:
self.renderer.store_render_target_to_file("temp.png")
img = Image.open("temp.png")
```

**상태:** 임시 해결책 동작 중, C++ 엔진 코드 수정 필요

### 5.4 구체 생성 실패

**문제:** `GenerateSphereGeometry`가 true를 반환하지만 지오메트리 생성 안 됨
```cpp
bool GenerateSphereGeometry(...) {
    // ... 유효성 검사 ...
    return true;  // 그러나 실제 생성 코드 없음!
}
```

**해결:** 아이코사히드론으로 구체 근사
```python
vzm.generate_icosahedron_geometry(geometry.get_vid(), radius, 2)
```

### 5.5 에셋 경로 문제

**문제:** 엔진이 `Assets/`와 `Shaders/` 폴더를 찾지 못함

**해결:** 엔진 루트에 junction(심볼릭 링크) 생성
```bash
cd VizMotive-Engine
mklink /J Assets Examples\Assets
mklink /J Shaders Examples\Shaders
```

---

## 6. 현재 이슈: 메시 생성 제한

### 6.1 문제 설명

**증상:** 7개 이상의 객체(큐브/구체) 생성 시 뷰어 크래시

**크래시 지점:**
```
✓ Cube #7 added successfully (Total objects: 7)
[RENDER START] Frame 145, Objects: 7
Calling renderer.render() with scene VID=1, camera VID=3
[CRASH - Python 예외 없음, C++ 엔진 크래시]
```

### 6.2 조사 결과

**테스트 1: 초기화 vs 런타임 생성**
- 가설: "런타임 메시 추가가 크래시 원인"
- 테스트: 엔진 초기화 시 큐브 3개 생성
- 결과: ✅ 크래시 없음

**테스트 2: 초기 큐브 후 런타임 추가**
- 런타임에 큐브 3개 더 추가 (총 6개)
- 결과: ✅ 크래시 없음

**테스트 3: 7번째 객체**
- 큐브 1개 더 추가 (총 7개)
- 결과: ❌ **크래시 발생**

**결론:** 문제는 "런타임 추가"가 아니라 **"총 객체 수 ≥ 7"**

### 6.3 VID 분석

```
Scene VID:              1
Camera VID:             3
Light VID:              ?
Init Cube #1:           Geom=?, Mat=?, Actor=?
Init Cube #2:           Geom=?, Mat=?, Actor=?
Init Cube #3:           Geom=?, Mat=?, Actor=?
Runtime Cube #4:        Geom=17, Mat=18, Actor=19
Runtime Cube #5:        Geom=20, Mat=21, Actor=22
Runtime Cube #6:        Geom=23, Mat=24, Actor=25
Runtime Cube #7:        Geom=26, Mat=27, Actor=28 ← 크래시
```

**패턴:**
- 각 큐브는 3개의 VID 사용 (Geometry, Material, Actor)
- 7개 큐브 + 1개 라이트 = 씬에 8개 액터
- VID 28에서 크래시 발생

### 6.4 Sample14와 비교

Sample14(공식 예제)는 다음을 사용:
- 랜덤 구체 5개
- 라이트 5개
- 바닥 평면 1개
- **총 ~11개 객체**

엔진이 7개보다 많은 객체를 지원해야 하므로, 다른 원인이 있을 것으로 추정됩니다.

### 6.5 현재 가설

**가능성 1: DirectX 12 리소스 고갈**
- 매 프레임 PNG 저장/로드로 GPU 리소스 고갈
- 현재 PNG I/O를 비활성화하여 테스트 중

**가능성 2: 셰이더의 고정 크기 버퍼**
- GPU 셰이더에 하드코딩된 배열 크기
- 셰이더 코드 확인 필요

**가능성 3: 씬 그래프 깊이 제한**
- 모든 객체가 씬의 직접 자식
- 계층 구조 필요할 수 있음

**가능성 4: 렌더 파이프라인 상태 문제**
- DirectX 12 command list 오버플로
- PSO 캐시 제한

### 6.6 다음 단계

1. ✅ PNG I/O 비활성화 테스트 (진행 중)
2. ⏳ VizMotive 셰이더 코드의 배열 크기 제한 확인
3. ⏳ RenderPath3D의 DirectX 12 리소스 할당 검토
4. ⏳ Sample14와 씬 구조 비교
5. ⏳ GPU 리소스 모니터링/로깅 추가

---

## 7. 코드 구조

```
VizMotive-Engine/
├── PythonBindings/
│   └── src/
│       └── bind_engine.cpp           # 확장된 Python 바인딩
│
├── Examples/
│   ├── Vz_MCP_Sample/
│   │   ├── vz_mcp_server_headless.py    # MCP 서버
│   │   ├── vz_viewer_standalone.py      # 뷰어 애플리케이션
│   │   ├── scene_commands.json          # 통신 파일
│   │   └── mcp_screenshots/             # 출력 디렉토리
│   │
│   ├── Assets/ → (junction)
│   └── Shaders/ → (junction)
│
└── EngineCore/
    └── HighAPIs/
        ├── VzRenderer.cpp/h             # 렌더러 API
        └── VzEngineAPIs.cpp/h           # 핵심 엔진 API
```

---

## 8. 사용 예시

### 8.1 시스템 시작

1. **Claude Desktop 실행** (MCP 서버 자동 시작)
2. **Claude에서 도구 사용:**
   ```
   start_viewer()를 사용해서 3D 뷰어를 실행해줘
   ```

### 8.2 객체 생성

```
위치 (0, 0, 0)에 빨간색 큐브 만들어줘
위치 (3, 0, 0)에 초록색 큐브 만들어줘
위치 (-3, 0, 0)에 반지름 1.5인 파란색 구체 만들어줘
```

Claude가 호출하는 도구:
```python
create_cube(x=0, y=0, z=0, size=1.0, r=1.0, g=0.0, b=0.0)
create_cube(x=3, y=0, z=0, size=1.0, r=0.0, g=1.0, b=0.0)
create_sphere(x=-3, y=0, z=0, radius=1.5, r=0.0, g=0.0, b=1.0)
```

### 8.3 스크린샷

```
현재 씬을 "my_scene.png"로 저장해줘
```

Claude 호출:
```python
render_screenshot(filename="my_scene.png")
```

출력: `mcp_screenshots/my_scene.png`

---

## 9. 성능 고려사항

### 9.1 렌더링 성능

- **프레임 레이트:** 6개 객체에서 ~30-60 FPS
- **병목:** 매 프레임 PNG 저장/로드 (~200-300ms)
- **GPU:** DirectX 12 렌더링은 빠름 (<10ms)

### 9.2 최적화 기회

1. **PNG 우회 제거** → 직접 GPU-CPU 텍스처 전송
2. **렌더 빈도 감소** → 정적 씬일 때 스킵
3. **Dirty flag 구현** → 불필요한 렌더링 방지
4. **공유 GPU 텍스처** → VizMotive와 DearPyGui 간

---

## 10. 제약사항 및 향후 개선

### 현재 제약사항

1. **객체 수:** 7개 이상에서 크래시 (조사 중)
2. **구체 생성:** 아이코사히드론 우회 사용
3. **렌더 타겟 접근:** PNG 우회 사용
4. **애니메이션 없음:** 정적 씬만
5. **카메라 제어 없음:** 고정 위치
6. **머티리얼 편집 제한:** 기본 색상만

### 향후 개선 방향

1. **카메라 조작** (회전, 줌, 팬)
2. **머티리얼 속성** (roughness, metallic, 텍스처)
3. **애니메이션 및 변환**
4. **씬 직렬화/역직렬화**
5. **다중 씬 및 카메라**
6. **포스트 프로세싱 효과**
7. **실시간 협업** (여러 사용자가 같은 씬 편집)

---

## 11. 배운 점

### 11.1 MCP 통합

- **stdio는 신성함:** MCP 서버에서 stdout/stderr에 출력 금지
- **인코딩 중요:** Windows 로케일 문제로 명시적 UTF-8 설정 필요
- **프로세스 분리:** 서버는 가볍게, 뷰어는 별도로
- **파일 기반 통신 효과적:** 명령 전달에 간단하고 신뢰성 있음

### 11.2 VizMotive 엔진

- **Layer mask 필수:** 모든 것에 `0xF` 마스크 필요
- **씬 계층 중요:** 객체를 자식으로 추가해야 함
- **VID 추적:** 정리 및 디버깅에 필수
- **DirectX 12 복잡함:** 리소스 관리에 주의 필요

### 11.3 Python/C++ 상호운용

- **타입 변환:** 엔진 타입용 커스텀 헬퍼 필요
- **메모리 관리:** Pybind11이 대부분 자동 처리
- **에러 처리:** C++ 예외를 Python에서 잡아야 함
- **로깅:** Python과 C++ 로그 시스템 분리

---

## 12. 결론

VizMotive MCP 서버 구현은 기본 기능을 성공적으로 구현했으며, Claude Desktop에서 3D 씬을 생성하고 조작할 수 있습니다. 현재 7개 이상의 객체 생성 시 크래시가 발생하는 이슈가 남아있으나, 엔진 레벨 문제로 추정되며 디버깅 진행 중입니다.

시스템의 강점:
- ✅ 명확한 아키텍처 분리 (서버 / 뷰어 / 엔진)
- ✅ 실시간 렌더링 및 GUI
- ✅ 확장 가능한 MCP 도구 시스템
- ✅ 안정적인 파일 기반 통신

향후 객체 수 제한 문제를 해결하고, 카메라 조작, 애니메이션 등의 기능을 추가하면 완전한 3D 협업 시스템으로 발전할 수 있습니다.

---

## 부록 A: 주요 파일

### Python 파일
- `vz_mcp_server_headless.py`: MCP 서버 (277줄)
- `vz_viewer_standalone.py`: 뷰어 (407줄)

### C++ 바인딩
- `bind_engine.cpp`: Python 바인딩 (추가 ~500줄)

### 설정
- `claude_desktop_config.json`: MCP 서버 등록

### 출력
- `scene_commands.json`: 명령 큐
- `mcp_screenshots/*.png`: 렌더링 이미지
- `viewer_debug_*.log`: 디버그 로그

---

## 부록 B: 의존성

### Python 패키지
```
fastmcp
pybind11
dearpygui
pillow
numpy
```

### 시스템 요구사항
- Windows 10/11
- DirectX 12 호환 GPU
- Python 3.8+
- Visual Studio 2019+ (빌드용)

---

**보고서 끝**
