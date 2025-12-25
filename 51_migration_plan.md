# Problem Statement
현재 repo는 Isaac Sim 4.2.x 시절의 `omni.isaac.*` 네임스페이스/확장(deprecated 예정)을 기준으로 작성되어 있습니다. Isaac Sim 5.0부터 deprecated 확장 지원이 제거되므로(= 5.1에서도 동일) 5.1 환경에서 extension 로딩/런타임 import가 깨질 가능성이 큽니다. citeturn7view1
# Current State (repo)
Isaac Sim 쪽(Extension)에서 직접 영향을 받는 부분은 주로 아래 파일입니다.
* `isaac.sim.mcp_extension/config/extension.toml`: dependency에 `omni.isaac.core`, `omni.isaac.ui` 사용 (4.5부터 rename 대상) citeturn1view3
* `isaac.sim.mcp_extension/isaac_sim_mcp_extension/extension.py`: `omni.isaac.*` import 다수(`World`, `XFormPrim`, `get_assets_root_path`, stage utils 등)
* `isaac.sim.mcp_extension/isaac_sim_mcp_extension/usd.py`: `add_reference_to_stage`를 `omni.isaac.core.utils.stage`에서 import
* `isaac.sim.mcp_extension/examples/*.py` 및 `isaac_mcp/server.py` docstring 예시 코드: `omni.isaac.*` import를 그대로 노출(사용자/에이전트가 복붙하면 5.1에서 실패)
MCP server(`isaac_mcp/server.py`) 자체는 Isaac Sim Python 모듈을 직접 import하지 않아서(대부분 소켓 통신) 영향은 “문서/예시 코드” 쪽이 더 큽니다.
# Target Definition
* “Isaac Sim 5.1에서 동작”: Isaac Sim 5.1에 extension을 enable 했을 때 import 에러 없이 로딩되고, MCP server를 통해 `get_scene_info`, `create_physics_scene`, `create_robot`, `execute_script` 기본 플로우가 정상 동작.
* (선택) “4.2/4.5/5.x 공존”: `try/except ImportError` 기반 shim으로 구버전도 계속 지원할지 여부를 결정.
# Proposed Changes (High-level)
## 1) 네임스페이스 변경 원칙 정리(Release notes 기반)
Isaac Sim 4.5에서 다수 `omni.isaac.*` 확장이 `isaacsim.*`로 rename 되었고, `omni.isaac.core`는 단일 패키지 → `isaacsim.core.api`, `isaacsim.core.prims`, `isaacsim.core.utils`로 분리됩니다. citeturn5view0turn7view1
또한 `omni.isaac.nucleus`는 `isaacsim.storage.native`로 이동하고, `omni.isaac.ui`는 `isaacsim.gui.components`로 이동합니다. citeturn1view3
(참고) `omni.isaac.wheeled_robots`는 `isaacsim.robot.wheeled_robots`로 rename. citeturn2view1
## 2) Extension dependency 업데이트
`isaac.sim.mcp_extension/config/extension.toml`의 `[dependencies]`를 5.1 기준으로 정리합니다.
* 제거/대체
    * `omni.isaac.core` → `isaacsim.core.api` + `isaacsim.core.prims` + `isaacsim.core.utils` citeturn5view0turn7view1
    * `omni.isaac.ui` → (실제로 UI를 쓰면) `isaacsim.gui.components`, UI를 안 쓰면 의존성에서 제거 citeturn1view3
    * `omni.isaac.nucleus`를 직접 dependency로 추가하지 않았더라도, 코드에서 `get_assets_root_path`를 쓰므로 제공 확장(`isaacsim.storage.native`) dependency를 명시적으로 추가하는 방향을 검토 citeturn1view3
* 유지
    * `omni.kit.uiapp`은 Kit 기본 구성요소라 그대로 두는 쪽이 안전(단, 실제로 UI를 전혀 안 쓰면 최소화 가능)
## 3) Python import 경로 마이그레이션
Extension 런타임 코드에서 `omni.isaac.*` → `isaacsim.*`로 변경합니다.
* 우선순위 1: 실제 extension 로딩에 필수인 파일
    * `isaac.sim.mcp_extension/isaac_sim_mcp_extension/extension.py`
    * `isaac.sim.mcp_extension/isaac_sim_mcp_extension/usd.py`
* 우선순위 2: 예시/문서(복붙 안전성)
    * `isaac.sim.mcp_extension/examples/*.py`
    * `isaac_mcp/server.py`의 docstring 예시 코드 블록
구체적으로 repo에서 많이 쓰는 심볼 기준 예상 매핑(초안)
* `World`, `SimulationContext`, `PhysicsContext`, `Articulation` 등 “코어 API” 성격 → `isaacsim.core.api ...`
* `XFormPrim` 등 prim wrapper → `isaacsim.core.prims ...`
* `add_reference_to_stage`, `is_stage_loading`, `create_prim` 등 utils → `isaacsim.core.utils.(stage|prims) ...`
* `get_assets_root_path` → `isaacsim.storage.native ...` (정확한 import 위치는 5.1 Python에서 확인 필요) citeturn1view3
중요: Isaac Sim 5.1에서의 실제 Python 모듈 경로는 “extension 이름과 1:1”이 아닐 수도 있으므로, 최종적으로는 Isaac Sim 5.1의 Python 콘솔에서 import 검증을 통해 확정합니다.
## 4) Dual-support(선택) shim 전략
만약 4.2/4.5 사용자도 계속 지원하고 싶다면, 다음과 같이 shim을 둡니다.
* 예: 
    * `try: from isaacsim.core.api import World; except ImportError: from omni.isaac.core import World`
    * `try: from isaacsim.core.utils.stage import add_reference_to_stage; except ImportError: from omni.isaac.core.utils.stage import add_reference_to_stage`
이 방식은 5.0+에서 구 네임스페이스가 완전히 제거된다는 전제에서(5.1 포함) “신 네임스페이스 우선”으로 작성해야 합니다. citeturn7view1
## 5) Kit 버전 상향(5.0/5.1)에서의 영향 점검
Isaac Sim 5.0/5.1은 Kit SDK가 107.x로 올라갑니다. citeturn6view0turn6view2
따라서 `omni.kit.async_engine.run_coroutine`, `omni.kit.commands`, `omni.usd` 등 Kit 레벨 API 사용 부는 동작은 하되, deprecation/behavior change가 없는지 smoke test로 확인합니다.
# Step-by-step Plan
## Phase 0: 기준선 확보
* Isaac Sim 5.1 환경에서 extension enable 시 현재 상태가 어떤 에러로 실패하는지(ImportError/extension dependency) 로그 확보
* “5.1만 지원” vs “4.2~5.1 동시 지원” 결정을 먼저 확정
## Phase 1: extension.toml dependency 교체
* `isaac.sim.mcp_extension/config/extension.toml`에서 deprecated dependency를 교체
    * `omni.isaac.core` 제거 → `isaacsim.core.api`, `isaacsim.core.prims`, `isaacsim.core.utils` 추가 citeturn5view0turn7view1
    * `omni.isaac.ui`는 실제 사용 여부에 따라 제거하거나 `isaacsim.gui.components`로 대체 citeturn1view3
    * `get_assets_root_path` 사용을 고려해 `isaacsim.storage.native` dependency 추가 citeturn1view3
## Phase 2: Python import 마이그레이션(필수 런타임 코드)
* `isaac.sim.mcp_extension/isaac_sim_mcp_extension/extension.py`:
    * `omni.isaac.nucleus.get_assets_root_path` → 5.1에서 유효한 경로로 변경(또는 shim)
    * `World`, `XFormPrim` 등의 import를 새 네임스페이스로 분리/이동
* `isaac.sim.mcp_extension/isaac_sim_mcp_extension/usd.py`:
    * `add_reference_to_stage` import 경로 변경(또는 shim)
## Phase 3: 예시/문서 코드 업데이트
* `isaac_mcp/server.py`의 docstring 예제 코드 블록에 있는 `omni.isaac.*` import를 `isaacsim.*`로 교체
* `isaac.sim.mcp_extension/examples/*.py`도 동일하게 교체
이 단계는 “실행”에는 필수는 아니지만, 5.1에서 사용자/에이전트가 예시를 그대로 실행할 수 있게 하는 품질/유지보수 포인트입니다.
## Phase 4: Isaac Sim 5.1에서 smoke test
* Isaac Sim 5.1을 extension enable로 실행
* MCP server 실행 후 다음 순서로 검증
    * `get_scene_info` 성공
    * `create_physics_scene` 성공
    * `create_robot`(franka/g1/go1 등) 성공
    * `usd.py` 로더가 stage에 reference 추가 가능한지(최소 1개 USD) 확인
    * `execute_script` 최소 코드(예: prim 1개 생성) 실행
* 실패 시 원인 분류
    * (A) extension dependency 누락 → extension.toml 수정
    * (B) import 경로 오탐 → shim/새 경로로 수정
    * (C) API behavior change(XFormPrim/World 등) → 5.1 API에 맞게 호출부 조정
## Phase 5: (선택) 4.x 호환성 유지
* Dual-support를 택한 경우, CI나 문서에 “지원 범위”를 명시하고, 최소한 import shim이 양쪽에서 동작하는지 확인
# Open Questions / Risks
* `get_assets_root_path`의 5.1에서의 정확한 Python import 위치는 release note에 “extension rename”만 제시되어 있어, 실제 모듈/심볼 위치는 5.1 Python에서 확인이 필요합니다. (플랜에서는 `isaacsim.storage.native` 기반으로 추정) citeturn1view3
* `isaacsim.core.prims`에서 `XFormPrim`의 구현/동작이 5.1에서 변경되었다고 언급되어 있어, `set_world_pose` 등의 동작이 기존과 동일한지 확인이 필요합니다. citeturn6view2
# Deliverables
* Isaac Sim 5.1에서 extension이 로딩되고 MCP 기본 툴 플로우가 동작하는 상태
* 변경된 dependency/namespace를 반영한 예시/문서 업데이트
* (선택) 4.2~5.1 공존을 위한 import shim
