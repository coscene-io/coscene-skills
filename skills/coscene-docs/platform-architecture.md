# coScene Platform Architecture

## Data Model Hierarchy

The coScene platform organizes data in a strict four-level hierarchy. Every
resource is scoped under exactly one parent — there are no cross-cutting
references or shared namespaces.

```
Organization (组织)
├── Project (项目)
│   ├── Record (记录)
│   │   ├── Files (multimodal: bags, MCAP, logs, images, video, config)
│   │   ├── Moments (一刻) — timestamped annotations on sensor timelines
│   │   ├── Labels (标签) — user-defined categorization tags
│   │   └── Custom Fields (自定义字段) — structured metadata
│   ├── Actions (动作) — automation building blocks (project-scoped)
│   ├── Triggers (触发器) — event-based action activation (project-scoped)
│   └── Rules (规则) — edge/device rule engine definitions (project-scoped)
├── Devices (设备) — org-level fleet, allocated to projects
├── Image Registry (镜像) — org-scoped container images for actions
└── Members — org-level users with project-level role assignments
```

### Organization (组织)

Top-level container — one per company or team. Owns all projects, members,
devices, container image registry, and billing. Org-level roles: Admin (full
control), Member (assigned project access), Viewer (read-only per visibility).

### Project (项目)

The RBAC boundary — membership and role determine read/write/admin access.
Contains records, actions, triggers, rules, and device allocations.

**Visibility:** Private (members only), Internal (org-wide), Public (anyone).

**Project roles:** Admin (manage project), Member (create/edit records, actions,
triggers, rules), Read-Only (view and download).

Actions and triggers are project-scoped — a trigger can only reference actions
within the same project. No cross-project trigger-to-action reference.

### Record (记录)

Core data container — NOT a single file, but a collection of all files
describing one scenario. Contains multimodal files (ROS bags, MCAP, logs, maps,
images, video), labels, custom fields, and moments. Records inherit project
visibility, can be copied or moved cross-project (requires permissions on both).

**States:** Active (read-write) or Archived (read-only, prevents modification).

### Moment (一刻)

Timestamped annotation marking a key time segment within a record — anchored to
sensor data timelines with precise start/end timestamps (NOT comments). Each
moment has name, description, start/end timestamps, and optional structured
attributes. Created manually during visualization playback or automatically by
rule engine triggers on the device edge (CEL expression match).

## Device Collector System

### coScout Edge Agent Architecture

coScout (端助理) is pre-built client software that runs on physical or logical
devices. It handles the full data lifecycle from capture to upload.

```
Device (Physical/Logical)
├── coScout Agent
│   ├── Directory Watcher
│   │   ├── listen_dirs — monitored for rule trigger evaluation
│   │   └── collect_dirs — source directories for data capture
│   ├── Topic Listener
│   │   └── topics — ROS/MCAP message streams for rule matching
│   ├── Rule Evaluator
│   │   ├── CEL expression matching on messages/events
│   │   ├── Event code table lookup (CSV/JSON)
│   │   ├── Time-based aggregation
│   │   └── Deduplication window (1-86400 seconds)
│   ├── Data Pipeline
│   │   ├── SHA256-based file deduplication
│   │   ├── Compression
│   │   ├── Filtering
│   │   └── Upload orchestration
│   └── Config Sync
│       └── Pulls rules from cloud every 60 seconds
└── Logs: ~/.local/state/cos/logs/cos.log
```

### Device Registration

Devices are registered at the organization level using a serial number (SN).
Registration methods:

- **SN file:** a file on the device containing its serial number
- **SN field:** a configuration field set during provisioning

Once registered, a device can be allocated to one or more projects. Device
allocation determines which project's rules are pushed to the device and where
collected data is uploaded.

### Data Flow

```
Device files/streams
       ↓
  coScout watches listen_dirs + collect_dirs + topics
       ↓
  Rule evaluation (CEL expressions, event codes)
       ↓  (trigger match)
  Data collection (time range before/after trigger event)
       ↓
  SHA256 dedup → compression → filtering
       ↓
  Upload to coScene cloud
       ↓
  Record created in target project (with labels from rule config)
       ↓
  Moments added at trigger timestamps
       ↓
  Visualization available in web UI
       ↓
  Triggers may fire (file upload, collection status change)
       ↓
  Actions execute (container or webhook)
```

### Configuration Parameters

| Parameter | Purpose | Example |
|---|---|---|
| `listen_dirs` | Directories monitored for rule trigger evaluation | `/data/ros_logs`, `/var/log/robot` |
| `collect_dirs` | Source directories for data capture on trigger | `/data/bags`, `/data/mcap` |
| `topics` | ROS/MCAP message streams for rule matching | `/diagnostics`, `/error_reports` |
| SN file/field | Device identity for registration | `/etc/cos/sn` or config field |

### Key Behaviors

- **SHA256 deduplication:** files with identical hashes are not re-uploaded,
  saving bandwidth and storage
- **60-second rule push interval:** the cloud pushes updated rules to edge
  devices every 60 seconds; there is no instant push mechanism
- **Deduplication window:** events matching the same rule within the configured
  window (1-86400 seconds) are merged to prevent duplicate collection
- **Variable interpolation:** rules support `{scope.*}` and `{msg.*}` syntax
  for dynamic record naming, labeling, and moment attributes

## Storage Model

All file data lives in an S3-compatible object store — users never interact
with S3 directly; access goes through the platform API or cocli.

Files are always scoped to a record (which is scoped to a project). There is no
project-level file storage outside of records. Each file has a path within the
record, SHA256 hash for deduplication, and size metadata.

Archived records become read-only — files, labels, and custom fields are
frozen. The record remains visible and downloadable. Unarchiving restores full
read-write access.

## Visualization

Browser-based visualization for replaying and analyzing record data:

- Timeline-based playback of ROS bags, MCAP files, and time-series data
- Multi-panel layout with configurable views (3D, image, plot, log, etc.)
- Synchronized playback across all panels
- Moment creation during playback — select a time range and annotate it

**Layouts** are JSON configuration files defining panel arrangement. Can be
created in the UI, exported as JSON, and imported into other records/projects.

**No CLI surface.** Visualization is web-only. cocli handles data operations;
visualization requires the browser.

## API Surface

coScene exposes supported platform operations through cocli and public OpenAPI.
Public skills should stay on those surfaces. If a capability is only visible in
internal service definitions or browser traffic and is not available through
cocli/public OpenAPI, direct the user to the platform UI instead of
reverse-engineering an API call.

### Endpoints

| Region | Endpoint | Notes |
|---|---|---|
| China | `openapi.coscene.cn` | Primary; zh docs at `docs.coscene.cn` |
| International | `openapi.coscene.io` | Mirror; en docs at `docs.coscene.io` |

### Authentication

All API access requires a bearer token:

- **`COS_TOKEN` env var** — used in automation actions (platform-injected,
  inherits trigger initiator permissions)
- **cocli profile** — `cocli login set -t <token>` stores in `~/.cocli.yaml`;
  `cocli login switch` toggles profiles

Tokens are generated in the web UI (User Settings). They carry the full
permissions of the creating user.

### Interaction Methods

1. **cocli** — official CLI for records, files, devices, projects
2. **Public OpenAPI** — supported public API operations when cocli does not cover them
3. **Platform UI** — action/trigger authoring, configuration, and visualization when no public API exists

## Permission Model

### Organization Roles

| Role | Capabilities |
|---|---|
| Admin | Full control: all projects, members, devices, images, billing |
| Member | Access assigned projects per project-level role |
| Viewer | Read-only per project visibility settings |

### Project Roles

| Role | Capabilities |
|---|---|
| Admin | Manage project: members, settings, all resources |
| Member | Create/edit: records, actions, triggers, rules, moments |
| Read-Only | View records, download files, run read-only operations |

### Device Allocation

Devices are registered at the org level. Org admins allocate devices to
projects — a device can belong to multiple projects. Only project members
see devices allocated to their project.

### Token Permission Inheritance

`COS_TOKEN` in action containers inherits the trigger initiator's full
permissions. If the initiator has Admin role, the token has Admin access. If
the initiator lacks target project permissions, cross-project calls fail 403.
Tokens are scoped to action lifetime — never log, exfiltrate, or persist them.
