# MVP Architecture Specification: PyQt + SimPy Logistics Modeling Desktop App

## 1. System Goals

### 1.1 Product goal
Build an MVP desktop application that lets industrial/logistics engineers model a discrete-event system visually, configure object behavior through properties, execute a SimPy-backed model, and inspect KPIs/results without writing code.

### 1.2 Primary design goals
- Desktop-first authoring using PyQt for docking, canvas interaction, and responsive editing.
- Deterministic discrete-event simulation using SimPy and an explicit random seed.
- Plant Simulation-inspired object modeling with reusable object types, property sheets, and directional connections.
- Immediate inspectability through event logs, KPIs, and optional runtime overlays on the canvas.
- JSON persistence for save/load, diffing, and future migration support.
- Safe runtime controls for run, pause, stop, and reset.

### 1.3 MVP non-goals
- 3D rendering.
- Multi-user collaboration.
- Optimization/experiment managers.
- Embedded scripting language support.
- Cloud/off-process execution.

### 1.4 Architectural quality attributes
- Modularity between UI, application orchestration, domain model, runtime, and persistence.
- Responsiveness while simulation is running.
- Determinism for reproducible runs.
- Extensibility for new object types.
- Debuggability via structured runtime events.
- Recoverability through explicit save/load/reset semantics.

## 2. User Workflows

### 2.1 Create a new model
1. User creates a new document.
2. Application initializes an empty `ModelDocument` with metadata and default canvas view state.
3. User drags `Source`, `Buffer`, `Processor`, or `Sink` objects from the left library onto the 2D canvas.
4. Canvas creates object instances with default properties.
5. User selects objects and edits their properties in the right panel.
6. User connects objects using a connect/link tool.
7. Validation feedback highlights missing or invalid configuration.

### 2.2 Edit an object
1. Canvas selection updates the active object.
2. Property panel builds a schema-driven editor for that object type.
3. User edits fields like name, capacity, processing time, or interarrival time.
4. Document store applies a typed command and marks the document dirty.
5. Canvas and validation indicators refresh.

### 2.3 Save and load
**Save**
1. User clicks Save.
2. Document is serialized to versioned JSON.
3. JSON is validated and written atomically.

**Load**
1. User opens a `.json` file.
2. Persistence service parses and validates the payload.
3. Migration logic upgrades old schema versions if necessary.
4. Document store replaces the in-memory document.
5. Canvas and inspectors rebind to the loaded model.

### 2.4 Run, pause, stop, and reset
**Run**
1. User clicks Run.
2. Controller validates the model.
3. A `SimulationSnapshot` is built from the editable document.
4. Worker thread starts the SimPy runtime.
5. Bottom panel begins receiving logs and KPI updates.

**Pause / Resume**
1. User clicks Pause.
2. Runtime enters a cooperative wait between event steps.
3. User clicks Run/Resume to continue.

**Stop / Reset**
1. User clicks Stop.
2. Runtime exits at the next safe checkpoint.
3. Runtime-only state is discarded.
4. Reset clears KPIs, logs, and runtime overlays while keeping the authored model intact.

## 3. Module Architecture

### 3.1 Architectural style
Use a layered, event-driven desktop architecture:
- **Presentation layer**: PyQt widgets, panels, canvas, action bindings.
- **Application layer**: controllers, document store, commands, selection, snapshot building.
- **Domain layer**: model objects, type descriptors, property schema, validation contracts.
- **Infrastructure layer**: SimPy runner, JSON persistence, metrics, logging, thread management.

### 3.2 Major modules
- `app`: bootstrap, dependency wiring, application startup.
- `ui.main_window`: shell window with toolbar and dock areas.
- `ui.object_library`: left palette of placeable object types.
- `ui.canvas`: `QGraphicsScene/QGraphicsView` editor for nodes and links.
- `ui.property_panel`: schema-driven form editor.
- `ui.bottom_panel`: tabs for logs, KPIs, and results.
- `application.document_store`: source of truth for current model and dirty state.
- `application.commands`: explicit document mutation commands.
- `application.controllers`: orchestration for document and simulation use cases.
- `domain.model`: `ModelDocument`, `SimObject`, `Connection`, view state, metadata.
- `domain.descriptors`: object library/type registration and property definitions.
- `services.validation_service`: document/object/connection validation.
- `simulation.snapshot`: immutable runtime snapshot.
- `simulation.runner`: worker-thread simulation execution.
- `simulation.nodes`: runtime implementations for `Source`, `Buffer`, `Processor`, `Sink`.
- `simulation.metrics`: KPI aggregation.
- `persistence.json_store`: save/load and schema version handling.

### 3.3 Dependency direction
Preferred dependency direction:
`ui -> application -> domain`
`application -> simulation / persistence / services`
`simulation -> domain`
`persistence -> domain`

Disallow:
- UI widgets directly mutating runtime internals.
- Worker thread directly touching Qt widgets.
- Persistence logic living inside widget classes.

## 4. Class Design

### 4.1 Domain classes
#### `ModelDocument`
- `model_id: str`
- `name: str`
- `schema_version: str`
- `metadata: ModelMetadata`
- `objects: dict[str, SimObject]`
- `connections: dict[str, Connection]`
- `view_state: CanvasViewState`

Responsibilities:
- Hold the authored model.
- Support lookup, serialization, and validation entry points.

#### `ModelMetadata`
- `author: str | None`
- `created_at: str`
- `updated_at: str`
- `description: str`
- `default_seed: int`
- `time_unit: str`
- `stop_condition: StopCondition`

#### `SimObject`
Use a generic instance model for MVP rather than a Python subclass per object instance.
- `id: str`
- `type_name: str`
- `name: str`
- `position: Point2D`
- `rotation: int`
- `properties: dict[str, Any]`

#### `Connection`
- `id: str`
- `source_object_id: str`
- `target_object_id: str`
- `source_port: str | None`
- `target_port: str | None`
- `routing_priority: int`

#### `ObjectTypeDescriptor`
- `type_name: str`
- `display_name: str`
- `category: str`
- `icon_path: str`
- `default_properties: dict[str, Any]`
- `property_definitions: list[PropertyDefinition]`
- `runtime_factory: type[RuntimeNodeFactory]`

#### `PropertyDefinition`
- `key: str`
- `label: str`
- `editor_type: str`
- `data_type: str`
- `default: Any`
- `required: bool`
- `min_value: float | None`
- `max_value: float | None`
- `enum_options: list[str]`
- `help_text: str`

### 4.2 Application classes
#### `DocumentStore`
- Holds the current `ModelDocument`.
- Applies commands.
- Tracks dirty state.
- Emits change notifications for UI refresh.

#### `CommandBus`
MVP commands:
- `AddObjectCommand`
- `MoveObjectCommand`
- `DeleteObjectCommand`
- `UpdatePropertyCommand`
- `AddConnectionCommand`
- `DeleteConnectionCommand`
- `LoadDocumentCommand`

#### `SelectionService`
Tracks selected object and connection IDs and notifies canvas/property views.

#### `DocumentController`
- `new_document()`
- `open_document(path)`
- `save_document(path)`
- `save_document_as(path)`

#### `SimulationController`
- `run()`
- `pause()`
- `resume()`
- `stop()`
- `reset()`
- `build_snapshot()`

#### `ValidationService`
- `validate_document(document) -> ValidationResult`
- `validate_object(sim_object, descriptor) -> list[ValidationIssue]`
- `validate_connections(document) -> list[ValidationIssue]`

### 4.3 UI classes
#### `MainWindow(QMainWindow)`
Hosts the toolbar and docked layout.

#### `ObjectLibraryWidget(QWidget)`
Shows registered descriptors and starts drag operations.

#### `CanvasView(QGraphicsView)`
Provides zoom, pan, and tool-mode handling.

#### `CanvasScene(QGraphicsScene)`
Maps document nodes and links to graphics items and handles drop/connect gestures.

#### `NodeGraphicsItem(QGraphicsItem)`
Renders a placed object and its runtime overlay state.

#### `ConnectionGraphicsItem(QGraphicsPathItem)`
Renders directional links.

#### `PropertyInspectorWidget(QWidget)`
Builds editor widgets from `PropertyDefinition` metadata.

#### `BottomPanelWidget(QTabWidget)`
Tabs:
- `EventLogWidget`
- `KpiWidget`
- `ResultsWidget`

### 4.4 Runtime classes
#### `SimulationSnapshot`
Immutable, validated copy of the current document used for execution.

#### `SimulationRunner(QObject)`
Worker-thread owner of `simpy.Environment` and runtime graph.

#### `RuntimeNode`
Base abstraction for runtime objects.
- `node_id`
- `env`
- `metrics`
- `upstream_refs`
- `downstream_refs`

Methods:
- `initialize()`
- `connect()`
- `start_processes()`
- `reset_metrics()`

#### `SourceNode`, `BufferNode`, `ProcessorNode`, `SinkNode`
Type-specific runtime implementations mapped from object descriptors.

#### `FlowEntity`
- `entity_id: str`
- `created_at: float`
- `attributes: dict[str, Any]`
- `trace: list[EntityTraceEvent]`

#### `RuntimeEvent`
- `timestamp: float`
- `event_type: str`
- `object_id: str | None`
- `entity_id: str | None`
- `payload: dict[str, Any]`

#### `MetricsCollector`
Consumes runtime events and computes KPI snapshots.

## 5. Data Schema

### 5.1 JSON envelope
```json
{
  "schema_version": "1.0",
  "model_id": "uuid",
  "name": "Inbound Line",
  "metadata": {
    "author": "Jane Doe",
    "created_at": "2026-03-22T10:00:00Z",
    "updated_at": "2026-03-22T10:30:00Z",
    "description": "Basic receiving flow",
    "default_seed": 42,
    "time_unit": "seconds",
    "stop_condition": {
      "mode": "until_time",
      "value": 28800
    }
  },
  "objects": [
    {
      "id": "obj_source_1",
      "type_name": "Source",
      "name": "Arrival Source",
      "position": {"x": 120, "y": 180},
      "rotation": 0,
      "properties": {
        "interarrival_time": 5.0,
        "batch_size": 1
      }
    }
  ],
  "connections": [
    {
      "id": "conn_1",
      "source_object_id": "obj_source_1",
      "target_object_id": "obj_buffer_1",
      "source_port": null,
      "target_port": null,
      "routing_priority": 1
    }
  ],
  "view_state": {
    "zoom": 1.0,
    "pan_x": 0,
    "pan_y": 0
  }
}
```

### 5.2 MVP object properties
- `Source`: `interarrival_time`, `first_creation_time`, `batch_size`, `max_creations`
- `Buffer`: `capacity`, `queue_discipline` (MVP supports `fifo` only)
- `Processor`: `processing_time`, `capacity`
- `Sink`: no required operational properties in MVP

### 5.3 Schema versioning strategy
- Include a top-level `schema_version`.
- Migrate older payloads on load.
- Prefer additive evolution and centralized migration functions.

## 6. Event Flow

### 6.1 Editor flow
**Object placement**
1. Drag begins in `ObjectLibraryWidget`.
2. Drop lands on `CanvasScene`.
3. Scene emits or dispatches `AddObjectCommand`.
4. `DocumentStore` updates the document.
5. Canvas re-renders and selects the new object.
6. Property inspector binds to it.

**Property edit**
1. User changes a property widget.
2. Property panel validates field shape.
3. `UpdatePropertyCommand` is dispatched.
4. Document store updates state.
5. Validation issues and dirty state refresh.

### 6.2 Simulation flow
**Run**
1. Toolbar action triggers `SimulationController.run()`.
2. Validation service performs preflight checks.
3. Snapshot builder creates an immutable runtime snapshot.
4. `SimulationRunner` starts on a `QThread`.
5. Runner emits status, clock, runtime events, and KPI snapshots.
6. Bottom panel and runtime overlays update on the UI thread.

**Pause**
1. Controller sets a pause request.
2. Runner checks the pause gate between `env.step()` calls.
3. Runner emits `PAUSED` status.

**Stop**
1. Controller sets a stop request.
2. Runner exits at the next safe checkpoint.
3. Final KPIs are emitted and runtime state is discarded.

### 6.3 Runtime event taxonomy
Minimum event types:
- `entity_created`
- `entity_entered_object`
- `entity_queued`
- `processing_started`
- `processing_finished`
- `entity_departed_object`
- `entity_disposed`
- `queue_length_changed`
- `simulation_started`
- `simulation_paused`
- `simulation_resumed`
- `simulation_stopped`
- `simulation_completed`
- `validation_error`

## 7. Threading Model

### 7.1 Thread split
Keep all PyQt widgets and editable document state on the main thread. Run the SimPy runtime in a dedicated worker `QThread`.

### 7.2 Concrete allocation
**Main/UI thread**
- `QApplication`
- `MainWindow`
- Canvas widgets/items
- Document store
- Controllers
- Property/KPI/log widgets

**Simulation worker thread**
- `SimulationRunner`
- `simpy.Environment`
- Runtime nodes
- Metrics collector

### 7.3 Communication model
Use queued Qt signals/slots for worker-to-UI communication.

Worker emits:
- `status_changed(status)`
- `log_event(RuntimeEvent)`
- `clock_updated(float)`
- `kpi_snapshot(KpiSnapshot)`
- `error(str)`
- `finished(RunResult)`

Controller/runner commands:
- `request_pause()`
- `request_resume()`
- `request_stop()`

### 7.4 Pause/stop mechanics
- Drive the simulation with repeated `env.step()` calls instead of a single blocking `env.run()`.
- After each step, check pause/stop flags.
- When paused, wait cooperatively until resume.
- Never allow worker code to access Qt widgets directly.

## 8. Persistence Design

### 8.1 Format and serializer
Use versioned JSON as the primary model format. Serializer responsibilities:
- Canonicalize field order where practical.
- Serialize by stable IDs.
- Preserve metadata and view state.
- Validate payload before write.

### 8.2 Atomic save flow
1. Serialize document to JSON text.
2. Validate against the schema.
3. Write to a temp file in the target directory.
4. Replace the destination file atomically.

### 8.3 Load flow
1. Read file.
2. Parse JSON.
3. Validate required structure.
4. Apply migrations if needed.
5. Construct `ModelDocument`.
6. Rebind UI to the loaded document.

### 8.4 Future extensions
- Autosave and crash recovery.
- Import/export reusable submodels.
- Bundled project assets.

## 9. Metrics Collection Design

### 9.1 MVP KPIs
**Global KPIs**
- simulation status
- simulated time reached
- total entities created
- total entities completed
- throughput rate
- average lead time
- current WIP

**Object KPIs**
- current queue length
- max queue length
- average queue length
- utilization for processors
- processed count

### 9.2 Measurement approach
Use event-driven accumulation:
- Runtime nodes emit entity lifecycle and state-change events.
- `MetricsCollector` updates accumulators incrementally.
- UI receives periodic `KpiSnapshot` updates.

### 9.3 Data structures
#### `KpiSnapshot`
- `sim_time: float`
- `global_metrics: dict[str, float | int | str]`
- `object_metrics: dict[str, ObjectKpiSnapshot]`

#### `ObjectKpiSnapshot`
- `object_id: str`
- `queue_length_current: int`
- `queue_length_max: int`
- `queue_length_avg: float`
- `utilization: float | None`
- `processed_count: int`

### 9.4 Time-weighted metrics
For averages such as queue length and utilization:
- record the last change timestamp
- accumulate `value * duration`
- divide by total simulated time when a snapshot is requested

## 10. MVP Scope vs Later Scope

### 10.1 MVP scope
- Single-document desktop app.
- `Source`, `Buffer`, `Processor`, `Sink` object types.
- 2D node/link editor.
- Left library, right property panel, bottom KPI/log/results panel.
- JSON save/load.
- Validation for common model errors.
- Run/pause/stop/reset controls.
- Deterministic seed-based runs.

### 10.2 Near-term next scope
- Undo/redo.
- Copy/paste/duplicate/delete.
- Routing policies.
- Entity attributes and branching logic.
- Charts in results panel.
- Rich runtime overlays on the canvas.

### 10.3 Later scope
- Hierarchical submodels.
- Conveyors/transporters/AGVs.
- Shifts, failures, and maintenance.
- Experiment manager and optimization loops.
- Embedded formulas/scripts.
- 3D or remote execution extensions.

## 11. Recommended File Tree

```text
op_2d_demo/
├── app/
│   ├── __init__.py
│   ├── bootstrap.py
│   ├── main.py
│   └── service_container.py
├── domain/
│   ├── __init__.py
│   ├── model.py
│   ├── metadata.py
│   ├── properties.py
│   ├── descriptors.py
│   ├── validation.py
│   └── events.py
├── application/
│   ├── __init__.py
│   ├── document_store.py
│   ├── commands.py
│   ├── selection_service.py
│   ├── document_controller.py
│   ├── canvas_controller.py
│   ├── simulation_controller.py
│   └── snapshot_builder.py
├── ui/
│   ├── __init__.py
│   ├── main_window.py
│   ├── actions.py
│   ├── object_library.py
│   ├── property_panel.py
│   ├── bottom_panel.py
│   ├── canvas/
│   │   ├── __init__.py
│   │   ├── scene.py
│   │   ├── view.py
│   │   ├── node_item.py
│   │   ├── connection_item.py
│   │   ├── tools.py
│   │   └── overlays.py
│   └── widgets/
│       ├── form_factory.py
│       ├── log_table.py
│       └── kpi_table.py
├── simulation/
│   ├── __init__.py
│   ├── runner.py
│   ├── snapshot.py
│   ├── entity.py
│   ├── runtime_base.py
│   ├── nodes/
│   │   ├── __init__.py
│   │   ├── source.py
│   │   ├── buffer.py
│   │   ├── processor.py
│   │   └── sink.py
│   ├── metrics.py
│   └── event_types.py
├── persistence/
│   ├── __init__.py
│   ├── json_store.py
│   ├── schema.py
│   └── migrations.py
├── services/
│   ├── __init__.py
│   ├── validation_service.py
│   ├── logging_service.py
│   └── id_service.py
├── assets/
│   ├── icons/
│   └── themes/
├── docs/
│   └── mvp_architecture_spec.md
├── tests/
│   ├── test_document_store.py
│   ├── test_validation_service.py
│   ├── test_json_store.py
│   ├── test_snapshot_builder.py
│   └── test_simulation_runner.py
├── pyproject.toml
├── README.md
└── .gitignore
```

## 12. Implementation Notes for Immediate Start

### 12.1 Suggested first sprint order
1. Define domain schema and descriptors.
2. Implement `DocumentStore` and command bus.
3. Build the `MainWindow` shell and panel layout.
4. Implement drag/drop object placement on the canvas.
5. Implement the property inspector.
6. Add JSON save/load.
7. Build snapshot creation and basic runtime nodes.
8. Add the worker-thread `SimulationRunner`.
9. Wire KPI/log output into the bottom panel.

### 12.2 Recommended reset semantics
Reset should:
- clear logs
- clear KPI snapshots
- remove runtime graph/overlays
- preserve the authored model exactly as edited

### 12.3 Recommended stop conditions for MVP
Support only:
- `until_time`
- `until_entities_completed`

### 12.4 Early technical risks to retire first
- Correct pause/stop behavior while stepping SimPy.
- UI responsiveness under high event throughput.
- Preventing mutable editor/runtime state leakage.
- Keeping property schemas extensible enough for future object types.

This specification is concrete enough for a team to begin implementing the domain model, controllers, PyQt shell, and simulation runtime in parallel.
