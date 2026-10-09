# AbilityFramework CRD/CR Architecture Evolution Design

[English](crd-architecture-evolution.md) | [简体中文](crd-architecture-evolution.zh-CN.md)

## Background

Today each ability package carries its own CRD file (`ability.crd.yaml` or `crds/*.crd.yaml`), which blurs the line between framework responsibilities and ability developer responsibilities. Following the k8s CRD/CR model:

- The **CRD** should be a resource type definition built into the framework (like a CRD provided by a k8s operator), defining the common structure of the resource
- The **CR** is the instance declaration written by the ability developer, describing the spec, interfaces, and configuration of a concrete ability

### Current Problems

1. **Blurred responsibilities** — the `AbilityCRD` struct mixes framework-level fields (ownership/bind modes, x-intentable, x-ref, FwkDetail) with ability-specific fields (provides, types, rpcMethods, tasks, config, debugOption, openAPIV3Schema)
2. **Redundant copies** — every ability package carries a CRD, yet the framework-level schema of the CRD is identical for all abilities
3. **Wrong concept in the toolchain** — openapi-tool's `generate-crd-from-openapi.py` names its output a "CRD", but its real purpose is generating a CR file conforming to the ability specification from an OpenAPI definition; the tool confuses CRD (framework specification) with CR (ability instance declaration)
4. **No Service type** — the framework only supports on-demand Abilities, not long-running Services

### Related Projects

| Project | Role |
|------|------|
| **abilityframework** | Core framework; manages ability deployment, execution, and monitoring |
| **openapi-tool** | Defines ability interfaces via OpenAPI YAML and generates CR files conforming to the CRD specification (`generate-crd-from-openapi.py`, currently misnamed as "generate CRD"); supports extensions such as `x-insightos`/`x-ability-config`/`x-tasks`/`x-depends` |
| **ability-py-sdk** | Python ability development SDK; provides the `AbilityInterface` base class (on_start/on_connect/on_disconnect/on_terminate), heartbeat management, and Flask IPC |
| **ability-sdk** | C++ ability development SDK; provides AbilityStub, AbilityConfig, TaskHelper, etc. |

---

## Phase 1: Internalize the CRD, Introduce the Manifest

### Core Idea

Extract the CRD from ability packages into a framework built-in definition. The ability-specific fields of the former CRD migrate to a new file `ability.manifest.yaml` (the ability manifest).

### Division of Responsibilities

```
Framework-maintained (CRD)            Written by ability developers (Manifest + CR)
┌─────────────────────┐           ┌─────────────────────────┐
│ ability.crd          │           │ ability.manifest.yaml    │
│ - CR structure schema│           │ - provides              │
│ - kind enum          │           │ - types, rpcMethods     │
│ - common spec fields │           │ - tasks                 │
│ - metadata rules     │           │ - config/status schema  │
│ - x-intentable semantics│        │ - depends               │
│ - x-ref semantics    │           │ - constants             │
└─────────────────────┘           └─────────────────────────┘
                                  ┌─────────────────────────┐
                                  │ my-ability.cr.yaml       │
                                  │ - kind: AtomAbility      │
                                  │ - metadata.name          │
                                  │ - spec.package/version   │
                                  │ - spec.config (values)   │
                                  │ - spec.devices           │
                                  └─────────────────────────┘
```

### CRD (framework built-in, single copy)

Embedded in the framework binary or loaded from `$ABILITY_FRAMEWORK_HOME/schema/ability.crd.yaml`:

```yaml
apiVersion: framework/v1
kind: CustomResourceDefinition
metadata:
  name: ability.crd
spec:
  names:
    kind: Ability           # covers AtomAbility, ComposeAbility, AbstractAbility
  schema:
    openAPIV3Schema:
      type: object
      required: [kind, metadata, spec]
      properties:
        kind:
          type: string
          enum: [AtomAbility, ComposeAbility, AbstractAbility]
        metadata:
          type: object
          required: [name]
          properties:
            name: { type: string }
            labels: { type: object, additionalProperties: { type: string } }
            annotations: { type: object, additionalProperties: { type: string } }
        spec:
          type: object
          required: [package, version, abilityName]
          properties:
            package: { type: string }
            version: { type: string }
            abilityName: { type: string }
            position: { type: string, default: "localhost" }
            autoStart: { type: boolean }
            priority: { type: integer }
            activityCondition: {}
            config: {}              # concrete schema defined by schema.config in the manifest
            debugOption: {}         # concrete schema defined by schema.debugOption in the manifest
            subabilities: { type: array }
            devices: { type: array }
            models: { type: array }
```

### Ability Manifest (carried in the package)

Written by the ability developer (can be generated via openapi-tool) and published with the package:

```yaml
abilityName: MyAbility.org
kind: AtomAbility
provides:                         # description of services the ability offers
  - name: ObjectDetection
    description: "Detect objects in images"
types: { ... }                    # custom type definitions
rpcMethods: { ... }               # RPC method signatures
tasks:                            # task definitions (maps to openapi-tool's x-tasks)
  - taskType: 0
    summary: "Run one detection"
    params:
      - name: timeout
        type: int
        optional: true
constants: { ... }                # constants exposed to the parent ability
schema:
  config:                         # OpenAPI V3 Schema validating CR spec.config
    openAPIV3Schema:
      type: object
      properties:
        threshold: { type: number, default: 0.5 }
  status:                         # Schema validating runtime status
    openAPIV3Schema: { ... }
  debugOption:                    # Schema validating CR spec.debugOption
    openAPIV3Schema:
      type: object
      properties:
        logPicture: { type: boolean, default: false }
depends:                          # dependency declarations
  abilities:
    - abilityName: Sub.Ability
      ownership: unique
      bind: local
  devices:
    - deviceName: Camera
      ownership: shared
  models:                         # model dependencies (maps to openapi-tool's x-depends.models)
    - spec:
        name: yolov5
        architecture: object_detection
        framework: pytorch
```

### Package Structure Change

```
packages/<name>/<version>/
  package.yaml               # package manifest (name, version, arch)
  ability.manifest.yaml      # ability manifest (NEW, replaces the former CRD)
  bin/
    ability                  # executable
```

### Validation Becomes Two-Level

1. **Framework-level validation** — the overall CR structure conforms to the built-in ability.crd schema
2. **Ability-level validation** — the CR's config/debugOption/status conforms to the schema in the manifest (x-intentable/x-ref expansion happens at this stage)

### openapi-tool Adaptation

`generate-crd-from-openapi.py` needs its concept corrected and its output adjusted:

- **Today (wrong concept)**: generates `*.crd.yaml` from an OpenAPI YAML, but what it actually generates is a CR, not a CRD
- **After Phase 1 (concept corrected)**:
  - The script is renamed to `generate-cr-from-openapi.py`, making clear that it generates a CR file
  - The output target becomes `*.cr.yaml` (a CR conforming to the ability.crd specification)
  - It also outputs `ability.manifest.yaml` (the ability manifest with interface descriptions, schemas, dependencies, etc.)
- The core conversion logic is unchanged (x-insightos → metadata, x-ability-config → spec.config, x-tasks → manifest.tasks, x-depends → manifest.depends); only the output format and field ownership change

### ability-py-sdk Adaptation

The SDK itself needs no major changes. The SDK currently initializes an ability from the CR information fetched via `/api/cr/{uuid}`; that flow is unchanged. Impact points:

- The CR structure returned by the framework is unchanged (CR fields are not modified)
- If the SDK needs to read ability interface descriptions (manifest content), add a `/api/manifest/{abilityName}/{version}` endpoint

### Key Code Changes

| File | Change |
|------|------|
| New `include/resourcemgr/ability_manifest.hpp` | AbilityManifest struct (extracted from AbilityCRD::Spec) |
| New `src/resourcemgr/ability_manifest.cpp` | JSON serialization, compatibility logic converting from the legacy CRD |
| `include/resourcemgr/ability_crd.hpp` | Slimmed down; keeps only the framework-level schema definition |
| `include/resourcemgr/ability_pkg.hpp` | AbilityPackage gains a manifests field |
| `src/resourcemgr/resource_mgr.cpp` | read_package() changed to read the manifest; the framework CRD is loaded once at startup |
| `src/resourcemgr/json_schema_utils.cpp` | Remove get_ability_crd_path(); validation becomes two-level (framework CRD + manifest) |
| `src/resourcemgr/resource_mgr_http_apis.cpp` | Adapt to the new validation path; add manifest query endpoint |
| `openapi-tool/generate-crd-from-openapi.py` | Renamed to `generate-cr-from-openapi.py`; outputs CR + manifest |

### Backward Compatibility

- If a package contains a legacy `ability.crd.yaml` or `crds/*.crd.yaml`, automatically extract the manifest fields to build an in-memory AbilityManifest
- Emit a deprecation warning log
- Legacy packages keep working without modification

---

## Phase 1.5: Ability Project Scaffolding Tool

### Core Idea

Combine openapi-tool and ability-py-sdk into a complete ability development toolchain:

1. **openapi-tool** generates the CR file (ability declaration) from an OpenAPI definition
2. **The scaffolding tool** reads the tasks definitions in the CR and, combined with SDK templates, automatically generates the ability project code

Developers only need to: define the OpenAPI interface → generate the CR → generate the project → **implement on_start() initialization plus each task function body**.

### Toolchain Flow

```
  OpenAPI YAML                   CR YAML                     Ability project
  (interface definition)         (ability declaration)       (runnable code)

  x-tasks:                       spec:                       task.py:
    - taskType: 0        ──>       tasks:               ──>    class DetectTask(TaskInterface):
      summary: Detect              - taskType: 0                def execute(self, input: dict) -> dict:
      params:                          params:                       timeout = input.get("timeout", 0)
        - name: timeout                  ...                          # TODO: implement detection logic
          type: int                                                   raise NotImplementedError
      returns:
        - name: result
          type: array

  ┌─────────────┐            ┌─────────────┐            ┌─────────────────────┐
  │ openapi-tool │    ──>    │ CR + Manifest│    ──>    │ scaffold-tool        │
  │ (define API) │            │ (generate decl)│         │ (generate project)   │
  └─────────────┘            └─────────────┘            └─────────────────────┘
```

### Scaffolded Project Structure

Example: a CR defining 3 tasks (taskType: 0=Detect, 1=Track, 2=Classify):

```
my-ability/
  main.py                          # entry point, auto-generated, usually no changes needed
  task.py                          # task implementations; developer fills in function bodies
  service/
    __init__.py                    # exports ImplAbility, task_manager
    ability.py                     # AbilityInterface implementation; developer fills in on_start
    server.py                      # Flask API server (copied from SDK template)
    interface.py                   # TaskInterface base class (copied from SDK template)
    task_manager.py                # TaskManager (copied from SDK template)
  ability.cr.yaml                  # CR generated by openapi-tool
  ability.manifest.yaml            # Manifest generated by openapi-tool
```

### Generated task.py Example

Suppose the CR defines:

```yaml
spec:
  tasks:
    - taskType: 0
      summary: "single recognition task"
      params:
        - name: timeout
          type: int
          optional: true
        - name: image_url
          type: string
      returns:
        - name: objects
          type: array
        - name: confidence
          type: number
```

The scaffold generates:

```python
from service import TaskInterface


class RecognizeTask(TaskInterface):
    """Single recognition task (taskType: 0)"""

    def execute(self, input_data: dict) -> dict:
        """
        Args:
            input_data: {
                "timeout": int (optional),
                "image_url": str,
            }

        Returns:
            {
                "objects": list,
                "confidence": float,
            }
        """
        timeout = input_data.get("timeout", 0)
        image_url = input_data.get("image_url", "")

        # TODO: implement recognition logic
        raise NotImplementedError("RecognizeTask.execute() is not implemented")
```

### Generated ability.py Example

```python
import ability_py
import threading
from .server import app
from werkzeug.serving import make_server


class ServerThread(threading.Thread):
    def __init__(self, host="0.0.0.0", port=8080):
        super().__init__()
        self.server = make_server(host, port, app)

    def run(self):
        self.server.serve_forever()

    def shutdown(self):
        self.server.shutdown()


class ImplAbility(ability_py.AbilityInterface):
    def __init__(self):
        self.ability_port = 0
        self.st = None

    def on_start(self):
        # TODO: initialize resources (load models, connect devices, etc.)
        pass

    def on_connect(self):
        if self.ability_port:
            return
        self.ability_port = ability_py.get_free_port()
        self.st = ServerThread(host="localhost", port=self.ability_port)
        self.st.start()

    def on_disconnect(self):
        if self.ability_port != 0:
            self.st.shutdown()
            self.ability_port = 0

    def on_terminate(self):
        # TODO: release resources
        pass

    def get_ability_port(self) -> int:
        return self.ability_port
```

### Generated main.py Example

```python
from service import ImplAbility, task_manager
import ability_py
from task import RecognizeTask  # imports auto-generated from CR tasks


# Register tasks (auto-generated from the taskType mapping in the CR)
tasks = {
    0: RecognizeTask(),
}

task_manager.register_tasks(tasks)

if __name__ == "__main__":
    ability = ImplAbility()
    service = ability_py.AbilityService()
    service.run(ability)
```

### Scaffolding Tool Commands

```bash
# Generate an ability project from a CR
python scaffold.py --cr ability.cr.yaml --output ./my-ability

# One step from OpenAPI: generate CR + project
python scaffold.py --openapi MyAbility.openapi.yaml --output ./my-ability
```

### Key Implementation

The scaffolding tool (`scaffold.py`) reads the tasks definitions in the CR/Manifest:

1. Generates one Task class per taskType (class name derived from summary or taskType)
2. Pre-fills parameter unpacking code in execute() (`input_data.get("param_name", default)`)
3. Lists the returns fields in the return-value docstring
4. Auto-registers all tasks in main.py
5. ability.py provides the standard lifecycle skeleton, with TODO left in on_start/on_terminate
6. The service/ directory is copied from SDK templates (server.py, interface.py, task_manager.py)

### Developers Only Focus On

1. **`on_start()` in `ability.py`** — initialization logic (loading models, connecting hardware, etc.)
2. **`execute()` of each Task in `task.py`** — business implementation (detection, planning, control, etc.)
3. **`on_terminate()` in `ability.py`** (optional) — resource release

Everything else (main.py entry, Flask server, TaskManager, heartbeats, lifecycle responses) is provided by the scaffold + SDK.

---

## Phase 2: New Service Resource Type

### Core Idea

Abilities are invoked on demand; Services run resident. The framework gains a built-in `service.crd` alongside `ability.crd`.

### Service vs. Ability

| Dimension | Ability | Service |
|------|---------|---------|
| Purpose | Run tasks on demand (e.g. object detection, path planning) | Provide resident services (e.g. video streaming, sensor acquisition) |
| Lifecycle | Inactive→Init→Standby→Running→Suspend→Terminated | Stopped→Starting→Running→Failed→Restarting |
| Exit behavior | Exit normally, notify parent ability | Auto-restart per restartPolicy |
| Restart policy | None | always / on-failure / never |
| Health check | Heartbeat timeout only (1-minute passive detection) | Active probing: HTTP liveness + readiness |
| SDK interface | on_start/on_connect/on_disconnect/on_terminate | on_start/on_stop + health endpoint |

### Service CRD (framework built-in)

```yaml
apiVersion: framework/v1
kind: CustomResourceDefinition
metadata:
  name: service.crd
spec:
  names:
    kind: Service
  schema:
    openAPIV3Schema:
      type: object
      required: [kind, metadata, spec]
      properties:
        kind:
          type: string
          enum: [Service]
        metadata:
          type: object
          required: [name]
          properties:
            name: { type: string }
            labels: { type: object, additionalProperties: { type: string } }
            annotations: { type: object, additionalProperties: { type: string } }
        spec:
          type: object
          required: [package, version, serviceName]
          properties:
            package: { type: string }
            version: { type: string }
            serviceName: { type: string }
            position: { type: string, default: "localhost" }
            config: {}
            restartPolicy:
              type: string
              enum: [always, on-failure, never]
              default: always
            maxRestarts: { type: integer, default: 5 }
            restartBackoffSeconds: { type: integer, default: 10 }
            healthCheck:
              type: object
              properties:
                liveness:
                  type: object
                  properties:
                    type: { type: string, enum: [http, process] }
                    path: { type: string }
                    port: { type: integer }
                    intervalSeconds: { type: integer, default: 10 }
                    failureThreshold: { type: integer, default: 3 }
                readiness:
                  type: object
                  properties:
                    type: { type: string, enum: [http] }
                    path: { type: string }
                    port: { type: integer }
                    intervalSeconds: { type: integer, default: 5 }
            devices: { type: array }
            models: { type: array }
```

### Service CR Example

```yaml
kind: Service
metadata:
  name: camera-stream
spec:
  package: com.robot.camera
  version: 1.0.0
  serviceName: CameraStream
  position: localhost
  restartPolicy: always
  maxRestarts: 5
  restartBackoffSeconds: 10
  healthCheck:
    liveness:
      type: http
      path: /health
      port: 9090
      intervalSeconds: 10
      failureThreshold: 3
  config:
    resolution: "1080p"
    fps: 30
```

### Service Package Structure

```
packages/<name>/<version>/
  package.yaml                # kind: service (new kind field to distinguish)
  service.manifest.yaml       # service manifest
  bin/
    service                   # executable
```

`package.yaml` gains a `kind` field, defaulting to `ability` for backward compatibility.

### Restart Policy Implementation

```
process exit -> on_service_exit() callback
  |-- restartPolicy == never -> mark Stopped, do not restart
  |-- restartPolicy == on-failure && exit_code == 0 -> mark Stopped, do not restart
  +-- otherwise -> check restartCount < maxRestarts
      |-- yes -> restart after backoff * 2^restartCount seconds, mark Restarting
      +-- no -> mark Failed, stop restarting
```

### ability-py-sdk Extension

New `ServiceInterface` base class:

```python
class ServiceInterface:
    def on_start(self):
        """Initialize and start the service"""
        pass

    def on_stop(self):
        """Stop the service"""
        pass

    def health_check(self) -> bool:
        """Return service health status; called periodically by the framework"""
        return True

    def get_service_port(self) -> int:
        """Return the service listening port"""
        return 0
```

### Key Code Changes

| File | Change |
|------|------|
| New `include/resourcemgr/service_cr.hpp` | ServiceCR, ServiceSpec structs |
| New `include/lifecyclemgr/service_state.hpp` | ServiceState enum (Stopped/Starting/Running/Failed/Restarting) |
| New `src/lifecyclemgr/service_lifecycle.cpp` | Restart policy implementation, health check probing logic |
| `src/lifecyclemgr/lifecycle_mgr.cpp` | on_service_exit() triggers restart decisions |
| `src/resourcemgr/resource_mgr.cpp` | ServiceCR create/delete/query, SQL_CREATE_SERVICE_CR_BASIC table |
| `src/resourcemgr/resource_mgr_http_apis.cpp` | New /api/service-cr, /api/service-heartbeat endpoints, etc. |
| `src/main.cpp` | Register health check timers |
| `ability-py-sdk/src/ability_py/` | New ServiceInterface, ServiceService |

### Backward Compatibility

Phase 2 is purely additive and does not affect any existing Ability behavior.

---

## Phase 3: Generalization and Unification

### 3.1 Resource Type Registry

Abstract a `ResourceTypeHandler` interface:

```cpp
class ResourceTypeHandler {
public:
    virtual ~ResourceTypeHandler() = default;
    virtual void validate_cr(const json& cr) = 0;
    virtual void on_start(const uuid& id) = 0;
    virtual void on_stop(const uuid& id) = 0;
    virtual void on_exit(const uuid& id, int exit_code) = 0;
    virtual void health_check(const uuid& id) = 0;
};
```

Ability and Service each implement one handler, looked up by kind. Future resource types only need to implement a handler + register a CRD.

### 3.2 Unified Package Format v3

```
packages/<name>/<version>/
  package.yaml          # name, version, arch, kind
  manifest.yaml         # unified manifest (ability/service distinguished by kind)
  bin/
    main                # unified entry point
```

### 3.3 Optional Ability keepAlive

Reuse the Phase 2 restart infrastructure to let Abilities declare keepAlive:

```yaml
kind: AtomAbility
spec:
  keepAlive:
    enabled: true
    maxRestarts: 3
    restartBackoffSeconds: 15
```

### 3.4 Legacy Format Removal Timeline

- Phase 3a: packages containing a legacy CRD emit a WARNING
- Phase 3b (next major version): remove the compatibility code that reads CRDs from packages
- Phase 3c: remove support for the v1 package format (crds/ directory)

---

## Implementation Order

```
Phase 1 (CRD internalization):
  1. Create the AbilityManifest struct and serialization
  2. Load the framework CRD as an embedded resource
  3. read_package() supports reading ability.manifest.yaml
  4. Legacy CRD -> Manifest compatibility bridge
  5. Refactor the validation flow (json_schema_utils.cpp)
  6. Update the HTTP API
  7. Update the DB tables
  8. Rename the openapi-tool script, fix the concept, output CR + manifest
  9. Integration tests

Phase 1.5 (ability project scaffolding):
  1. scaffold.py core logic: read tasks definitions from CR/Manifest
  2. Task class code generation (parameter unpacking, return-value comments, TODO placeholders)
  3. ability.py lifecycle skeleton generation
  4. main.py entry generation (auto task registration)
  5. service/ template directory copied from the SDK
  6. --openapi mode: integrate openapi-tool for one-step generation
  7. End-to-end test: OpenAPI → CR → project → framework loads and runs it

Phase 2 (Service resource type):
  1. ServiceCR/ServiceSpec structs
  2. Service CRD embedding
  3. ServiceState enum and lifecycle logic
  4. Restart policy + health checks
  5. Service HTTP API
  6. Service DB tables
  7. package.yaml kind field
  8. ability-py-sdk ServiceInterface
  9. Integration tests

Phase 3 (generalization):
  1. ResourceTypeHandler abstraction
  2. Unified package format
  3. Ability keepAlive
  4. Legacy format deprecation path
```

## Verification

### Phase 1
1. New-format packages (with ability.manifest.yaml) load correctly and validate CRs
2. Legacy-format packages (with ability.crd.yaml) convert correctly through the compatibility bridge
3. Old and new validation paths produce identical results
4. `GET /api/crd` returns the framework-level CRD
5. openapi-tool (after renaming) correctly outputs CR + manifest formats

### Phase 1.5
1. Generate a project from an example CR; confirm the directory structure and files are complete
2. The parameter unpacking code in the generated task.py matches the CR tasks definitions
3. After filling in task execute(), the project loads and runs under the framework
4. --openapi mode end to end: OpenAPI YAML → generated project → framework starts the ability → task API call succeeds

### Phase 2
1. Create a Service CR, verify the process starts
2. Kill the process; it auto-restarts per restartPolicy
3. After maxRestarts is reached, it is marked Failed
4. A failed health check probe triggers a restart
5. restartPolicy: never verifies exit without restart
6. ability-py-sdk ServiceInterface works correctly
