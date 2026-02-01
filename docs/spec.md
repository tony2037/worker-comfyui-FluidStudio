# Technical Specification: worker-comfyui-FluidStudio

## Table of Contents
- [1. Project Overview](#1-project-overview)
- [2. System Architecture](#2-system-architecture)
- [3. Component Design](#3-component-design)
- [4. Request Flow](#4-request-flow)
- [5. Data Storage Architecture (No Database)](#5-data-storage-architecture-no-database)
- [6. Data Structures](#6-data-structures)
- [7. API Specification](#7-api-specification)
- [8. Integration Points](#8-integration-points)
- [9. Error Handling](#9-error-handling)
- [10. Configuration](#10-configuration)
- [11. Deployment Architecture](#11-deployment-architecture)
- [12. Performance & Scalability](#12-performance--scalability)
- [13. Security Considerations](#13-security-considerations)

---

## 1. Project Overview

### 1.1 Purpose
`worker-comfyui-FluidStudio` is a **serverless wrapper for ComfyUI** that transforms the interactive ComfyUI UI into a robust, production-ready API service on the RunPod platform. It enables users to execute ComfyUI workflows via HTTP API calls, receiving generated images as base64 strings or S3 URLs.

### 1.2 Key Features
- **Serverless Architecture**: Scalable, pay-per-use execution on RunPod infrastructure
- **Workflow Execution**: Submit ComfyUI workflows in JSON format via API
- **Real-time Monitoring**: WebSocket-based progress tracking with automatic reconnection
- **Flexible Output**: Base64-encoded images (default) or S3 upload mode
- **Image Input**: Support for base64-encoded input images
- **Comprehensive Error Handling**: Detailed validation and error reporting
- **Network Volume Support**: Persistent model storage across worker instances
- **Multiple Deployment Options**: Pre-built Docker images, custom builds, or network volumes
- **Development Support**: Local API simulation and debugging tools

### 1.3 Technology Stack
- **Language**: Python 3.12
- **Runtime**: Ubuntu 24.04 with CUDA 12.6.3/12.8.1
- **Core Framework**: ComfyUI (AI workflow engine)
- **Serverless Platform**: RunPod SDK (`runpod~=1.7.12`)
- **Communication**: WebSocket Client, Requests
- **Container**: Docker with multi-stage builds
- **GPU**: NVIDIA GPUs with CUDA support

### 1.4 Current Branch: FluidStudio-Qwen

This branch is a **production-focused customization** of worker-comfyui v5.3.0, optimized for network volume-based model management.

**Key Characteristics:**
- **Default Model Type**: `ztex` (no pre-downloaded models)
- **Network Volume**: **Mandatory** (not optional) for ztex deployments
- **Build Time**: 5-10 minutes (vs 30-60 minutes for model-included variants)
- **Docker Image Size**: ~5-7 GB (vs 15-30 GB for model-included variants)
- **Test Suite**: Removed for production focus (commits: `3349260`, `aeb6bdf`)

**Architectural Changes:**
1. **Model Loading**: All models loaded from `/runpod-volume/models/` (network volume)
2. **Directory Structure**: Added `models/loras/` and explicit `models/unet/` configuration
3. **Build Process**: All `wget` model download commands removed (commit: `559eb7a`)
4. **File Structure**: Test directories removed (`tests/`, `test_resources/`, `.runpod/tests.json`)

**When to Use This Branch:**
- Production deployments requiring flexible model management
- Scenarios where models change frequently
- Multi-tenant environments sharing models
- Fast build/deployment cycles critical

**For comprehensive details**, see [FluidStudio-Qwen Guide](fluidstudio-qwen.md).

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        RunPod Platform                           │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Serverless Worker Container                   │ │
│  │                                                            │ │
│  │  ┌──────────────┐         ┌──────────────────────────┐   │ │
│  │  │  RunPod SDK  │────────▶│    handler.py            │   │ │
│  │  │  (Listener)  │         │  - Input validation      │   │ │
│  │  └──────────────┘         │  - Image upload          │   │ │
│  │         │                 │  - Workflow queue        │   │ │
│  │         │                 │  - WebSocket monitor     │   │ │
│  │         │                 │  - Output retrieval      │   │ │
│  │         ▼                 └──────────┬───────────────┘   │ │
│  │  ┌──────────────┐                   │                    │ │
│  │  │  API Client  │                   │ HTTP/WS            │ │
│  │  │  /run        │◀──────────────────┘                    │ │
│  │  │  /runsync    │         ┌──────────────────────────┐   │ │
│  │  │  /status     │         │      ComfyUI Server      │   │ │
│  │  │  /health     │         │  127.0.0.1:8188          │   │ │
│  │  └──────────────┘         │  - Workflow engine       │   │ │
│  │                           │  - Model loading         │   │ │
│  │  ┌──────────────┐         │  - Image generation      │   │ │
│  │  │ S3 Uploader  │         └──────────────────────────┘   │ │
│  │  │ (Optional)   │                   │                    │ │
│  │  └──────────────┘                   │                    │ │
│  │         │                           │                    │ │
│  │         └───────────────────────────┘                    │ │
│  │  ┌──────────────────────────────────────────────────┐   │ │
│  │  │           Network Volume (Optional)               │   │ │
│  │  │           /runpod-volume/models/                  │   │ │
│  │  └──────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
         │                                              │
         ▼                                              ▼
  ┌─────────────┐                              ┌─────────────┐
  │   Client    │                              │  AWS S3     │
  │   (API)     │                              │  (Optional) │
  └─────────────┘                              └─────────────┘
```

### 2.2 Component Layers

#### Layer 1: API Gateway (RunPod Platform)
- Receives HTTP requests from clients
- Routes to serverless worker instances
- Manages job queue and worker scaling
- Returns job IDs or synchronous results

#### Layer 2: Handler (handler.py)
- Validates and processes input
- Orchestrates workflow execution
- Monitors progress via WebSocket
- Retrieves and formats output

#### Layer 3: ComfyUI Engine
- Loads and executes workflows
- Manages GPU resources
- Generates images
- Provides HTTP/WebSocket APIs

#### Layer 4: Storage
- **Temporary**: In-container filesystem for images
- **Persistent**: Network volumes for models
- **Output**: S3 (optional) or base64 response

### 2.3 Process Model
- **Stateless Workers**: Each job can run on a fresh worker instance
- **Single Job per Worker**: Worker processes one job at a time
- **Auto-scaling**: RunPod manages worker pool based on load
- **Optional Refresh**: `REFRESH_WORKER=true` stops worker after each job

---

## 3. Component Design

### 3.1 handler.py (Main Orchestrator)

**File**: `handler.py` (836 lines)

#### 3.1.1 Core Functions

| Function | Lines | Purpose | Key Operations |
|----------|-------|---------|----------------|
| `handler()` | 507-831 | Main entry point | Job orchestration, error handling |
| `validate_input()` | 142-188 | Input validation | Workflow structure, images array |
| `check_server()` | 191-224 | Health check | ComfyUI availability retry loop |
| `upload_images()` | 227-308 | Image upload | Base64 decode, multipart POST |
| `queue_workflow()` | 340-452 | Workflow submission | HTTP POST to /prompt endpoint |
| `get_history()` | 455-468 | Result retrieval | Fetch execution history |
| `get_image_data()` | 471-504 | Image download | Fetch from /view endpoint |
| `_attempt_websocket_reconnect()` | 71-139 | Reconnection logic | Health check + reconnect |

#### 3.1.2 WebSocket Monitoring

**Connection Setup** (lines 594-605):
```python
client_id = str(uuid.uuid4())
ws = websocket.WebSocket()
ws.connect(f"ws://{COMFY_HOST}/ws?clientId={client_id}")
```

**Message Handling** (lines 606-656):
- **Type "status"**: Queue position updates
- **Type "executing"**: Node execution progress
- **Type "execution_error"**: Workflow errors with details
- **Completion Detection**: `node=null` and matching `prompt_id`

**Reconnection Strategy**:
1. Detect WebSocket closure
2. Check ComfyUI HTTP status (immediate fail if server crashed)
3. Reconnect with exponential backoff
4. Resume monitoring from current state
5. Maximum 5 attempts with 3s delay

#### 3.1.3 Output Processing

**Image Retrieval Flow** (lines 686-774):
```
For each output node:
  1. Parse output data structure
  2. Filter out temporary images (type='temp')
  3. Fetch image bytes via GET /view
  4. If S3 configured:
     - Write to temp file
     - Upload via rp_upload.upload_image()
     - Return {"filename", "type": "s3_url", "data": url}
  5. Else:
     - Encode as base64
     - Return {"filename", "type": "base64", "data": b64_string}
```

### 3.2 network_volume.py (Diagnostics)

**File**: `src/network_volume.py` (154 lines)

**Purpose**: Optional diagnostics tool for troubleshooting model path issues.

**Activation**: Set `NETWORK_VOLUME_DEBUG=true` environment variable.

**Functionality**:
- Validates `/runpod-volume` mount existence
- Checks `extra_model_paths.yaml` configuration
- Scans model directories for expected file types
- Reports file counts, sizes, and directory structure
- Provides helpful error messages with expected layout

**Output Example**:
```
=== Network Volume Diagnostics ===
Mount point: /runpod-volume - EXISTS
Config file: /comfyui/extra_model_paths.yaml - EXISTS

Found models:
  checkpoints/ - 3 files (12.5 GB)
    - model_v1.safetensors (4.2 GB)
    - model_v2.ckpt (8.3 GB)
  loras/ - 15 files (450 MB)
  vae/ - 2 files (800 MB)
===================================
```

### 3.3 Startup Script (start.sh)

**File**: `src/start.sh` (26 lines)

**Execution Flow**:
```bash
1. Load libtcmalloc for memory optimization
2. Set ComfyUI-Manager to offline mode (security)
3. Start ComfyUI server in background
4. If SERVE_API_LOCALLY=true:
     Start local RunPod API simulation
   Else:
     Start RunPod handler (production)
```

**ComfyUI Launch Command**:
```bash
python /comfyui/ComfyUI/main.py \
  --listen 127.0.0.1 \
  --port 8188 \
  --log-level $COMFY_LOG_LEVEL
```

### 3.4 Configuration Files

#### 3.4.1 extra_model_paths.yaml
Maps network volume to ComfyUI model directories:

```yaml
runpod_worker_comfy:
  base_path: /runpod-volume
  checkpoints: models/checkpoints/
  clip: models/clip/
  clip_vision: models/clip_vision/
  configs: models/configs/
  controlnet: models/controlnet/
  embeddings: models/embeddings/
  loras: models/loras/
  upscale_models: models/upscale_models/
  vae: models/vae/
```

**Effect**: ComfyUI automatically discovers models at these paths without requiring them to be baked into the Docker image.

#### 3.4.2 requirements.txt
```
runpod~=1.7.12          # RunPod serverless SDK
websocket-client        # WebSocket communication
requests                # HTTP client
```

### 3.5 Utility Scripts

#### 3.5.1 comfy-node-install.sh
**Purpose**: Safe wrapper around `comfy node install` command.

**Problem Solved**: `comfy-cli` returns exit code 0 even on installation failures.

**Solution**:
- Captures installation output
- Parses for error indicators:
  - "No such file or directory"
  - "ImportError"
  - "ModuleNotFoundError"
  - "RuntimeError"
- Returns proper exit code
- Suggests checking registry.comfy.org

**Usage**:
```bash
comfy-node-install.sh comfyui-kjnodes comfyui-ic-light
```

#### 3.5.2 comfy-manager-set-mode.sh
**Purpose**: Configure ComfyUI-Manager network mode.

**Modes**:
- `public`: Full internet access
- `private`: Restricted access
- `offline`: No network access (container default)

**Implementation**:
- Modifies `/comfyui/user/default/ComfyUI-Manager/config.ini`
- Sets `security_level` parameter
- Creates config if missing

---

## 4. Request Flow

### 4.1 Detailed Execution Sequence

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. CLIENT REQUEST                                               │
│    POST /runsync or /run                                        │
│    {                                                            │
│      "input": {                                                 │
│        "workflow": {...},                                       │
│        "images": [{name, image_b64}, ...],                      │
│        "comfy_org_api_key": "..."                               │
│      }                                                          │
│    }                                                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. RUNPOD PLATFORM                                              │
│    - Route request to worker                                    │
│    - Assign job_id                                              │
│    - Scale workers if needed                                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. HANDLER: INITIALIZATION                                      │
│    handler(job) called                                          │
│    - Extract job_id                                             │
│    - Get input data                                             │
│    - Optional: Network volume diagnostics                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. INPUT VALIDATION (validate_input)                            │
│    ✓ Check workflow exists and is dict                          │
│    ✓ Validate images array structure                            │
│    ✓ Validate each image has 'name' and 'image'                 │
│    ✓ Extract comfy_org_api_key (optional)                       │
│                                                                 │
│    On failure: Return {"error": "...", "details": [...]}        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. SERVER HEALTH CHECK (check_server)                           │
│    Retry loop (max 500 attempts, 50ms interval):                │
│      GET http://127.0.0.1:8188/                                 │
│      If 200 OK: Continue                                        │
│      If timeout: Retry                                          │
│                                                                 │
│    On failure: Return {"error": "Server not reachable"}         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. IMAGE UPLOAD (upload_images) - If images provided            │
│    For each image:                                              │
│      1. Strip data URI prefix (data:image/png;base64,)          │
│      2. Decode base64 to bytes                                  │
│      3. Create multipart form:                                  │
│         - image: (name, bytes, content-type)                    │
│      4. POST http://127.0.0.1:8188/upload/image                 │
│      5. Validate response contains 'name' field                 │
│                                                                 │
│    Partial failures: Continue with successful uploads           │
│    Total failure: Return error with details                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. WEBSOCKET CONNECTION SETUP                                   │
│    client_id = uuid4()                                          │
│    ws = websocket.WebSocket()                                   │
│    ws.connect("ws://127.0.0.1:8188/ws?clientId={client_id}")    │
│                                                                 │
│    Configure:                                                   │
│      - WEBSOCKET_TRACE for debugging (optional)                 │
│      - Timeout settings                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. QUEUE WORKFLOW (queue_workflow)                              │
│    Prepare payload:                                             │
│      - client_id                                                │
│      - prompt: workflow                                         │
│      - extra_data: {                                            │
│          api_key_comfy_org: comfy_org_api_key (if provided)     │
│        }                                                        │
│                                                                 │
│    POST http://127.0.0.1:8188/prompt                            │
│                                                                 │
│    Success: Extract prompt_id                                   │
│    400 Error: Parse validation errors (node issues, missing     │
│               models, etc.) and return detailed error           │
│    Other Error: Return generic error                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 9. MONITOR EXECUTION (WebSocket Loop)                           │
│    While True:                                                  │
│      message = ws.recv()                                        │
│      data = json.loads(message)                                 │
│                                                                 │
│      If type == "status":                                       │
│        - Log queue position                                     │
│                                                                 │
│      If type == "executing":                                    │
│        - data['node'] = current node ID                         │
│        - data['prompt_id'] = workflow ID                        │
│        - If node == null AND prompt_id matches:                 │
│            WORKFLOW COMPLETE - Exit loop                        │
│                                                                 │
│      If type == "execution_error":                              │
│        - Extract node_id, node_type, exception                  │
│        - Return error with details                              │
│                                                                 │
│      On WebSocketConnectionClosedException:                     │
│        - Call _attempt_websocket_reconnect()                    │
│        - If success: Continue monitoring                        │
│        - If failure: Return error                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 10. FETCH EXECUTION HISTORY (get_history)                       │
│     GET http://127.0.0.1:8188/history/{prompt_id}               │
│                                                                 │
│     Extract:                                                    │
│       - outputs: Dict of node outputs                           │
│       - Each output contains:                                   │
│         * images: [{"filename", "subfolder", "type"}, ...]      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 11. PROCESS OUTPUT IMAGES                                       │
│     For each output node:                                       │
│       For each image in node outputs:                           │
│         1. Skip if type == 'temp' (temporary)                   │
│         2. Call get_image_data(filename, subfolder, type)       │
│         3. Receive raw image bytes                              │
│                                                                 │
│         If S3 configured (BUCKET_ENDPOINT_URL set):             │
│           a. Write to temp file                                 │
│           b. rp_upload.upload_image(job_id, temp_path)          │
│           c. Delete temp file                                   │
│           d. Append: {                                          │
│                "filename": filename,                            │
│                "type": "s3_url",                                │
│                "data": s3_url                                   │
│              }                                                  │
│                                                                 │
│         Else (base64 mode):                                     │
│           a. Encode bytes to base64                             │
│           b. Append: {                                          │
│                "filename": filename,                            │
│                "type": "base64",                                │
│                "data": base64_string                            │
│              }                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 12. RETURN RESPONSE                                             │
│     Success:                                                    │
│       {                                                         │
│         "images": [                                             │
│           {"filename", "type", "data"},                         │
│           ...                                                   │
│         ]                                                       │
│       }                                                         │
│                                                                 │
│     With non-fatal warnings:                                    │
│       {                                                         │
│         "images": [...],                                        │
│         "errors": ["warning1", "warning2"]                      │
│       }                                                         │
│                                                                 │
│     Complete failure:                                           │
│       {                                                         │
│         "error": "Error description",                           │
│         "details": ["detail1", "detail2"]                       │
│       }                                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 13. CLIENT RECEIVES RESPONSE                                    │
│     /runsync: Immediate response with result                    │
│     /run: Job ID returned, poll /status/{job_id}                │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Timing Characteristics

| Phase | Typical Duration | Notes |
|-------|------------------|-------|
| Input validation | <10ms | In-memory checks |
| Server health check | 50ms - 25s | Max 500 retries × 50ms |
| Image upload | 100ms - 2s/image | Depends on image size |
| Workflow queue | 50-200ms | Network + validation |
| Workflow execution | 2s - 60s+ | Depends on model, GPU |
| History fetch | 50-100ms | Small JSON response |
| Image download | 100ms - 1s/image | Depends on resolution |
| S3 upload | 200ms - 3s/image | Network dependent |
| Base64 encoding | 50-200ms/image | CPU-bound |

### 4.3 WebSocket Reconnection Flow

```
1. WebSocket connection closed detected
   ↓
2. Check ComfyUI HTTP health: GET /
   ↓
3. If HTTP fails:
     Server crashed → Return error immediately
   ↓
4. If HTTP succeeds:
     Server alive, WS issue → Attempt reconnect
   ↓
5. Reconnect with same client_id
   ↓
6. Success: Resume monitoring from current state
   ↓
7. Failure: Wait WEBSOCKET_RECONNECT_DELAY_S seconds
   ↓
8. Retry (max WEBSOCKET_RECONNECT_ATTEMPTS times)
   ↓
9. If all attempts fail: Return error
```

---

## 5. Data Storage Architecture (No Database)

### 5.1 Stateless Design Philosophy

**worker-comfyui-FluidStudio** follows a **fully stateless, database-free architecture**. This design decision is fundamental to the serverless nature of the worker.

### 5.2 No Database Components

**What is NOT used:**
- ❌ No SQL databases (PostgreSQL, MySQL, SQLite, etc.)
- ❌ No NoSQL databases (MongoDB, Redis, DynamoDB, etc.)
- ❌ No embedded databases or key-value stores
- ❌ No persistent job queues or result storage
- ❌ No session management or state tracking

**Why this matters:**
- ✅ **True serverless**: Workers can start/stop without state synchronization
- ✅ **Horizontal scaling**: No database connection limits or locking
- ✅ **Instant deployment**: No migrations or schema management
- ✅ **Lower cost**: No database infrastructure to maintain
- ✅ **Simpler operations**: No backups, replication, or failover

### 5.3 Data Storage Mechanisms

#### 5.3.1 Job State (RunPod-Managed)
**How it works:**
- RunPod platform manages job queue and state
- Workers receive jobs via RunPod SDK
- Workers return results to RunPod
- RunPod handles job persistence, retries, and status tracking

**What the worker stores:** Nothing. Job state is ephemeral in worker memory.

#### 5.3.2 Models (File-Based)
**Storage locations:**
```
Option 1: Baked into Docker image
  /comfyui/models/checkpoints/model.safetensors

Option 2: Network volume (ztex branch default)
  /runpod-volume/models/checkpoints/model.safetensors
```

**Persistence:** Files are read-only from worker perspective.

#### 5.3.3 Generated Images (Temporary)
**Storage flow:**
```
1. ComfyUI generates image → /tmp/<uuid>.png
2. Worker downloads image → memory (bytes)
3. Worker encodes to base64 OR uploads to S3
4. Worker returns result to RunPod
5. Worker terminates → all temp files deleted
```

**Persistence:** None. Images exist only during job execution.

#### 5.3.4 Logs (Ephemeral)
**Logging:**
- Stdout/stderr captured by RunPod platform
- No log files written to disk
- No log aggregation database

**Persistence:** RunPod retains logs per their retention policy.

#### 5.3.5 Configuration (Environment Variables)
**Storage:**
- Environment variables set at deployment time
- No runtime configuration changes
- No config file updates

**Persistence:** Immutable per worker instance.

### 5.4 Comparison: Database vs File-Based

| Aspect | Traditional (Database) | worker-comfyui (File-Based) |
|--------|----------------------|---------------------------|
| **Job State** | Stored in DB tables | RunPod platform handles |
| **Models** | Stored in DB or blob storage | Files on disk/volume |
| **Generated Images** | Stored in DB or blob storage | Temporary, then base64/S3 |
| **Configuration** | Stored in DB | Environment variables |
| **User Data** | Stored in DB | Not applicable (stateless) |
| **Logs** | Stored in DB or log aggregator | Stdout (RunPod captures) |
| **Scaling** | Limited by DB connections | Unlimited (stateless) |
| **Deployment** | Migrations required | Push image and run |
| **Cost** | DB infrastructure + compute | Compute only |

### 5.5 Why No Database Works

**Serverless execution model:**
```
Request arrives → Worker starts → Process job → Return result → Worker stops
                  (no state)                    (no state)     (no state)
```

**Key enablers:**
1. **Idempotent operations**: Each job is self-contained
2. **External state management**: RunPod handles persistence
3. **File-based models**: Models are static resources
4. **Temporary outputs**: Results returned immediately, not stored
5. **No user sessions**: Each request is independent

**When you WOULD need a database:**
- Multi-step workflows with long-running state
- User accounts and authentication
- Job history and analytics
- Rate limiting and quotas
- Inter-job dependencies

**Why worker-comfyui doesn't need these:**
- RunPod handles authentication
- RunPod provides analytics
- Jobs are independent
- No multi-user state

### 5.6 Data Flow Without Database

```
Client Request
   ↓
RunPod Platform (manages queue)
   ↓
Worker Instance (stateless)
   ↓
Load models from file system
   ↓
Execute workflow (in-memory state)
   ↓
Generate images (temporary files)
   ↓
Encode/upload images
   ↓
Return result to RunPod
   ↓
Worker terminates (no state saved)
```

**No database interaction at any step.**

---

## 6. Data Structures

### 6.1 Request Schema

#### 6.1.1 Complete Request Object
```json
{
  "input": {
    "workflow": {
      "<node_id>": {
        "inputs": {
          "<input_name>": "<value>" | ["<node_id>", <output_index>],
          ...
        },
        "class_type": "<node_class>",
        "_meta": {
          "title": "<node_title>"
        }
      },
      ...
    },
    "images": [
      {
        "name": "input_image.png",
        "image": "data:image/png;base64,iVBORw0KGg..." | "iVBORw0KGg..."
      },
      ...
    ],
    "comfy_org_api_key": "optional-api-key"
  }
}
```

#### 6.1.2 Workflow Object Structure
ComfyUI workflows are node graphs where:
- **Node ID**: Unique string/number identifier
- **inputs**: Key-value pairs of node parameters
  - Direct values: strings, numbers, booleans
  - Node references: `["<source_node_id>", <output_slot>]`
- **class_type**: ComfyUI node type (e.g., "CLIPTextEncode", "KSampler")
- **_meta**: Optional metadata (title, etc.)

**Example Node**:
```json
"6": {
  "inputs": {
    "text": "a photo of a cat",
    "clip": ["30", 1]
  },
  "class_type": "CLIPTextEncode",
  "_meta": {
    "title": "Positive Prompt"
  }
}
```

#### 5.1.3 Images Array Schema
```typescript
interface ImageInput {
  name: string;        // Filename to reference in workflow
  image: string;       // Base64 encoded image (with or without data URI)
}
```

**Valid Formats**:
- With prefix: `"data:image/png;base64,iVBORw0KGg..."`
- Without prefix: `"iVBORw0KGg..."`

### 5.2 Response Schema

#### 5.2.1 Success Response
```json
{
  "id": "sync-uuid-string",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "base64" | "s3_url",
        "data": "<base64_string>" | "<s3_url>"
      }
    ]
  },
  "delayTime": 123,
  "executionTime": 4567
}
```

#### 5.2.2 Partial Success Response (with warnings)
```json
{
  "id": "sync-uuid-string",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "base64",
        "data": "..."
      }
    ],
    "errors": [
      "Failed to upload image2.png to S3: Connection timeout",
      "Image3.png not found in workflow output"
    ]
  },
  "delayTime": 123,
  "executionTime": 4567
}
```

#### 5.2.3 Error Response
```json
{
  "id": "sync-uuid-string",
  "status": "FAILED",
  "error": "Workflow validation failed",
  "details": [
    "Node '10' (KSampler): Required input 'model' not connected",
    "Node '12' (SaveImage): Invalid filename parameter"
  ]
}
```

#### 5.2.4 Validation Error Response (400)
```json
{
  "error": "Workflow validation failed",
  "details": [
    "Node 10 (KSampler):",
    "  - Required input 'model' is not provided",
    "  - Input 'steps' must be an integer between 1 and 10000",
    "Node 15 (CheckpointLoaderSimple):",
    "  - Model 'model_name.safetensors' not found",
    "  Available models:",
    "    - sd_xl_base_1.0.safetensors",
    "    - flux1-dev.safetensors"
  ]
}
```

### 5.3 Internal Data Structures

#### 5.3.1 WebSocket Message Types

**Status Message**:
```json
{
  "type": "status",
  "data": {
    "status": {
      "exec_info": {
        "queue_remaining": 2
      }
    }
  }
}
```

**Executing Message**:
```json
{
  "type": "executing",
  "data": {
    "node": "12" | null,
    "prompt_id": "uuid-string"
  }
}
```
*Note: `node=null` with matching `prompt_id` indicates completion*

**Execution Error Message**:
```json
{
  "type": "execution_error",
  "data": {
    "prompt_id": "uuid-string",
    "node_id": "15",
    "node_type": "KSampler",
    "exception_message": "CUDA out of memory",
    "exception_type": "RuntimeError",
    "traceback": ["...", "..."]
  }
}
```

#### 5.3.2 ComfyUI History Response
```json
{
  "prompt_id": {
    "prompt": [1, "uuid", {...}],
    "outputs": {
      "9": {
        "images": [
          {
            "filename": "ComfyUI_00001_.png",
            "subfolder": "",
            "type": "output"
          }
        ]
      }
    },
    "status": {
      "status_str": "success",
      "completed": true,
      "messages": [[...], [...]]
    }
  }
}
```

### 5.4 Configuration Data Structures

#### 5.4.1 extra_model_paths.yaml Schema
```yaml
<profile_name>:
  base_path: string           # Base directory path
  checkpoints: string         # Relative path to checkpoints
  clip: string                # Relative path to CLIP models
  clip_vision: string         # Relative path to CLIP vision models
  configs: string             # Relative path to configs
  controlnet: string          # Relative path to ControlNet models
  embeddings: string          # Relative path to embeddings
  loras: string               # Relative path to LoRAs
  upscale_models: string      # Relative path to upscale models
  vae: string                 # Relative path to VAE models
  unet: string                # Relative path to UNet models
```

---

## 6. API Specification

### 6.1 RunPod Endpoints

#### 6.1.1 POST /run (Asynchronous)
**Purpose**: Queue a job and return immediately with job ID.

**Request**:
```http
POST https://api.runpod.ai/v2/<endpoint_id>/run
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "input": {
    "workflow": {...},
    "images": [...],
    "comfy_org_api_key": "..."
  }
}
```

**Response**:
```json
{
  "id": "job-uuid",
  "status": "IN_QUEUE"
}
```

**Size Limit**: 10 MB

#### 6.1.2 POST /runsync (Synchronous)
**Purpose**: Wait for job completion and return result directly.

**Request**: Same as `/run`

**Response**: Complete job result (see [5.2 Response Schema](#52-response-schema))

**Timeout**: Configurable per endpoint (default: 300s)
**Size Limit**: 20 MB

#### 6.1.3 GET /status/{job_id}
**Purpose**: Check status of asynchronous job.

**Request**:
```http
GET https://api.runpod.ai/v2/<endpoint_id>/status/<job_id>
Authorization: Bearer <api_key>
```

**Response** (in progress):
```json
{
  "id": "job-uuid",
  "status": "IN_PROGRESS"
}
```

**Response** (completed):
```json
{
  "id": "job-uuid",
  "status": "COMPLETED",
  "output": {...}
}
```

#### 6.1.4 GET /health
**Purpose**: Check endpoint health.

**Response**:
```json
{
  "status": "healthy",
  "workers": {
    "ready": 5,
    "running": 3
  }
}
```

### 6.2 ComfyUI Internal API

#### 6.2.1 GET /
**Purpose**: Health check endpoint.

**Response**: HTTP 200 with HTML page.

#### 6.2.2 GET /object_info
**Purpose**: Get available nodes and models.

**Response**:
```json
{
  "KSampler": {
    "input": {
      "required": {
        "model": ["MODEL"],
        "steps": ["INT", {"default": 20, "min": 1, "max": 10000}],
        ...
      }
    },
    "output": ["LATENT"]
  },
  ...
}
```

#### 6.2.3 POST /upload/image
**Purpose**: Upload input images.

**Request**:
```http
POST http://127.0.0.1:8188/upload/image
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="image"; filename="test.png"
Content-Type: image/png

<binary_data>
--boundary--
```

**Response**:
```json
{
  "name": "test.png",
  "subfolder": "",
  "type": "input"
}
```

#### 6.2.4 POST /prompt
**Purpose**: Queue workflow execution.

**Request**:
```json
{
  "client_id": "uuid",
  "prompt": {
    "6": {...},
    "10": {...}
  },
  "extra_data": {
    "api_key_comfy_org": "..."
  }
}
```

**Response** (success):
```json
{
  "prompt_id": "uuid",
  "number": 42,
  "node_errors": {}
}
```

**Response** (validation error - 400):
```json
{
  "error": {
    "type": "prompt_error",
    "message": "Validation failed",
    "details": "...",
    "extra_info": {}
  },
  "node_errors": {
    "10": {
      "errors": [
        {
          "type": "required",
          "message": "Required input 'model' is missing",
          "details": "..."
        }
      ],
      "dependent_outputs": [],
      "class_type": "KSampler"
    }
  }
}
```

#### 6.2.5 GET /history/{prompt_id}
**Purpose**: Get workflow execution results.

**Response**: See [5.3.2 ComfyUI History Response](#532-comfyui-history-response)

#### 6.2.6 GET /view
**Purpose**: Download generated images.

**Query Parameters**:
- `filename`: Image filename
- `subfolder`: Subfolder path (optional)
- `type`: Image type ("output", "input", "temp")

**Response**: Binary image data

#### 6.2.7 WebSocket ws://127.0.0.1:8188/ws
**Purpose**: Real-time workflow execution updates.

**Query Parameters**:
- `clientId`: UUID for client identification

**Messages**: See [5.3.1 WebSocket Message Types](#531-websocket-message-types)

---

## 7. Integration Points

### 7.1 RunPod Platform Integration

#### 7.1.1 SDK Usage
```python
import runpod

# Handler function receives jobs
def handler(job):
    # job['id'] - Job identifier
    # job['input'] - Input data
    return {"images": [...]}

# Start listening for jobs
runpod.serverless.start({"handler": handler})
```

#### 7.1.2 Image Upload to S3
```python
from runpod.serverless.utils import rp_upload

# Upload image to S3
url = rp_upload.upload_image(job_id, file_path)
# Returns: https://bucket.s3.region.amazonaws.com/job_id/filename.ext
```

#### 7.1.3 Network Volumes
- **Mount Path**: `/runpod-volume` (serverless) or `/workspace` (pods)
- **Configuration**: Set via RunPod console when creating template
- **Persistence**: Shared across all worker instances
- **Use Case**: Store large models without baking into Docker

### 7.2 AWS S3 Integration

#### 7.2.1 Configuration
Required environment variables:
- `BUCKET_ENDPOINT_URL`: Full S3 bucket URL
- `BUCKET_ACCESS_KEY_ID`: AWS access key
- `BUCKET_SECRET_ACCESS_KEY`: AWS secret key

#### 7.2.2 Upload Path Structure
```
s3://<bucket>/
  <job_id>/
    ComfyUI_00001_.png
    ComfyUI_00002_.png
    ...
```

#### 7.2.3 Required IAM Permissions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::<bucket>/*"
    }
  ]
}
```

### 7.3 Comfy.org API Integration

#### 7.3.1 API Key Configuration
- **Global**: Set `COMFY_ORG_API_KEY` environment variable
- **Per-request**: Include `comfy_org_api_key` in input

#### 7.3.2 Key Injection
The handler transforms the key for ComfyUI:
```python
# Input
comfy_org_api_key = input.get("comfy_org_api_key")

# Injected into workflow payload
extra_data = {
    "api_key_comfy_org": comfy_org_api_key  # Note: transformed key name
}
```

#### 7.3.3 Usage in Workflows
ComfyUI API nodes automatically use the injected key for:
- Model downloads from Comfy.org
- API-based nodes requiring authentication

---

## 8. Error Handling

### 8.1 Error Categories

#### 8.1.1 Client Errors (4xx - User Fixable)

| Error Type | Cause | Response Structure |
|------------|-------|-------------------|
| Missing Input | No workflow provided | `{"error": "Please provide input"}` |
| Invalid JSON | Malformed workflow | `{"error": "Invalid JSON format"}` |
| Invalid Images | Wrong image array structure | `{"error": "...", "details": [...]}` |
| Workflow Validation | ComfyUI validation failure | `{"error": "...", "details": [node errors]}` |
| Missing Model | Model not found | `{"error": "...", "details": [available models]}` |
| Invalid Parameters | Node parameter out of range | `{"error": "...", "details": [constraints]}` |

#### 8.1.2 Server Errors (5xx - System Issues)

| Error Type | Cause | Response Structure |
|------------|-------|-------------------|
| Server Unreachable | ComfyUI not responding | `{"error": "Server not reachable"}` |
| WebSocket Closed | Connection dropped | Automatic reconnection |
| Execution Error | Workflow runtime error | `{"error": "...", "details": [exception]}` |
| S3 Upload Failed | Network/auth issue | Warning in `errors` array |
| CUDA OOM | Insufficient GPU memory | `{"error": "CUDA out of memory"}` |

### 8.2 Error Handling Strategies

#### 8.2.1 Retry Mechanisms

**Server Health Check** (`handler.py:191-224`):
```python
max_retries = 500
retry_interval_ms = 50

for i in range(max_retries):
    try:
        response = requests.get(f"http://{COMFY_HOST}/", timeout=5)
        if response.status_code == 200:
            return True
    except requests.RequestException:
        time.sleep(retry_interval_ms / 1000)
        continue

return False
```

**WebSocket Reconnection** (`handler.py:71-139`):
```python
max_attempts = int(os.getenv("WEBSOCKET_RECONNECT_ATTEMPTS", "5"))
delay_seconds = int(os.getenv("WEBSOCKET_RECONNECT_DELAY_S", "3"))

for attempt in range(1, max_attempts + 1):
    # Check HTTP health first
    if not check_server():
        return None  # Server crashed, fail immediately

    # Attempt reconnect
    try:
        ws.connect(f"ws://{COMFY_HOST}/ws?clientId={client_id}")
        return ws
    except Exception:
        time.sleep(delay_seconds)

return None
```

#### 8.2.2 Partial Failure Handling

**Image Upload** (`handler.py:227-308`):
- Continue processing successful uploads even if some fail
- Collect errors in list
- Return partial success with error details

**Output Processing** (`handler.py:686-774`):
- Skip temporary images
- Continue on individual image fetch failures
- Log errors but don't fail entire job
- Return available images with warnings

#### 8.2.3 Graceful Degradation

**S3 Upload Fallback**:
```python
try:
    # Attempt S3 upload
    url = rp_upload.upload_image(job_id, temp_path)
    images.append({"filename": filename, "type": "s3_url", "data": url})
except Exception as e:
    # Fall back to base64
    logging.error(f"S3 upload failed: {e}")
    errors.append(f"Failed to upload {filename} to S3")
    # Continue with other images
```

### 8.3 Error Response Format

#### 8.3.1 Detailed Validation Error
```python
{
    "error": "Workflow validation failed",
    "details": [
        "Node 10 (KSampler):",
        "  - Required input 'model' is not provided",
        "  - Input 'steps' must be an integer between 1 and 10000",
        "",
        "Node 15 (CheckpointLoaderSimple):",
        "  - Model 'nonexistent.safetensors' not found",
        "  Available models:",
        "    - sd_xl_base_1.0.safetensors",
        "    - flux1-dev.safetensors"
    ]
}
```

#### 8.3.2 Execution Error with Context
```python
{
    "error": "Workflow execution failed at node 12 (KSampler)",
    "details": [
        "Node Type: KSampler",
        "Exception: RuntimeError",
        "Message: CUDA out of memory. Tried to allocate 2.00 GiB",
        "Traceback:",
        "  File 'nodes.py', line 123, in sample",
        "  ..."
    ]
}
```

### 8.4 Logging Strategy

#### 8.4.1 Log Levels
```python
logging.DEBUG   # WebSocket frames, detailed state
logging.INFO    # Job start/complete, major steps
logging.WARNING # Partial failures, recoverable errors
logging.ERROR   # Request failures, unrecoverable errors
```

#### 8.4.2 Structured Logging
```python
logger.info(f"Job {job_id}: Processing workflow with {len(workflow)} nodes")
logger.info(f"Job {job_id}: Uploading {len(images)} input images")
logger.info(f"Job {job_id}: Workflow queued with prompt_id {prompt_id}")
logger.info(f"Job {job_id}: Execution complete, processing {image_count} outputs")
logger.error(f"Job {job_id}: Workflow validation failed", exc_info=True)
```

---

## 9. Configuration

### 9.1 Environment Variables

#### 9.1.1 Complete Configuration Reference

| Variable | Type | Default | Purpose | Validation |
|----------|------|---------|---------|------------|
| **General** |
| `REFRESH_WORKER` | boolean | `false` | Stop worker after each job | `true`/`false` |
| `SERVE_API_LOCALLY` | boolean | `false` | Enable local API server | `true`/`false` |
| `COMFY_ORG_API_KEY` | string | – | Global Comfy.org API key | Any string |
| **Logging** |
| `COMFY_LOG_LEVEL` | string | `DEBUG` | ComfyUI log verbosity | `DEBUG`/`INFO`/`WARNING`/`ERROR`/`CRITICAL` |
| `NETWORK_VOLUME_DEBUG` | boolean | `false` | Enable volume diagnostics | `true`/`false` |
| **WebSocket** |
| `WEBSOCKET_RECONNECT_ATTEMPTS` | integer | `5` | Max reconnection attempts | 1-100 |
| `WEBSOCKET_RECONNECT_DELAY_S` | integer | `3` | Delay between attempts (seconds) | 1-60 |
| `WEBSOCKET_TRACE` | boolean | `false` | Enable frame-level tracing | `true`/`false` |
| **S3 Upload** |
| `BUCKET_ENDPOINT_URL` | string | – | S3 endpoint URL (enables S3) | Valid URL |
| `BUCKET_ACCESS_KEY_ID` | string | – | AWS access key | Required if S3 enabled |
| `BUCKET_SECRET_ACCESS_KEY` | string | – | AWS secret key | Required if S3 enabled |

#### 9.1.2 Configuration Validation
```python
# S3 configuration check
bucket_endpoint = os.environ.get("BUCKET_ENDPOINT_URL")
if bucket_endpoint:
    # S3 mode enabled
    bucket_access_key = os.environ.get("BUCKET_ACCESS_KEY_ID")
    bucket_secret = os.environ.get("BUCKET_SECRET_ACCESS_KEY")
    if not bucket_access_key or not bucket_secret:
        raise ValueError("S3 credentials incomplete")
```

### 9.2 Hardcoded Constants

#### 9.2.1 handler.py Configuration
```python
# ComfyUI server configuration
COMFY_HOST = "127.0.0.1:8188"
COMFY_API_AVAILABLE_INTERVAL_MS = 50
COMFY_API_AVAILABLE_MAX_RETRIES = 500

# WebSocket configuration
DEFAULT_WEBSOCKET_RECONNECT_ATTEMPTS = 5
DEFAULT_WEBSOCKET_RECONNECT_DELAY_S = 3
```

#### 9.2.2 Why These Are Hardcoded
- **COMFY_HOST**: ComfyUI always runs locally in container
- **Retry intervals**: Tuned for typical startup times
- **Defaults**: Reasonable for most use cases, overridable via env vars

### 9.3 Docker Build Arguments

#### 9.3.1 Dockerfile ARGs

| Argument | Purpose | Default | Example |
|----------|---------|---------|---------|
| `BASE_IMAGE` | CUDA base image | `nvidia/cuda:12.6.3...` | `nvidia/cuda:12.8.1...` |
| `COMFYUI_VERSION` | ComfyUI version | Latest | `0.3.67` |
| `CUDA_VERSION_FOR_COMFY` | CUDA version for comfy-cli | `12.6` | `12.8` |
| `ENABLE_PYTORCH_UPGRADE` | Upgrade PyTorch | `false` | `true` |
| `PYTORCH_INDEX_URL` | PyTorch index URL | – | `https://download.pytorch.org/whl/cu128` |
| `MODEL_TYPE` | Model variant to download | `base` | `sdxl`/`sd3`/`flux1-dev`/`ztex` |
| `HUGGINGFACE_ACCESS_TOKEN` | HF token for gated models | – | `hf_...` |

#### 9.3.2 docker-bake.hcl Targets

```hcl
target "flux1-dev" {
  dockerfile = "Dockerfile"
  contexts = {
    scripts = "scripts"
    src = "src"
  }
  target = "final"
  tags = ["runpod/worker-comfyui:dev-flux1-dev"]
  args = {
    MODEL_TYPE = "flux1-dev"
    BASE_IMAGE = "nvidia/cuda:12.6.3-cudnn-runtime-ubuntu24.04"
    COMFYUI_VERSION = "0.3.67"
  }
}
```

### 9.4 ComfyUI Configuration

#### 9.4.1 Startup Arguments
```bash
python /comfyui/ComfyUI/main.py \
  --listen 127.0.0.1 \
  --port 8188 \
  --log-level $COMFY_LOG_LEVEL
```

#### 9.4.2 ComfyUI-Manager Config
```ini
[default]
security_level=offline
channel_url=https://raw.githubusercontent.com/ltdrdata/ComfyUI-Manager/main
```

**Modes**:
- `public`: Full internet access (development)
- `private`: Restricted access
- `offline`: No network (production/container)

---

## 10. Deployment Architecture

### 10.1 Deployment Models

#### 10.1.1 Pre-built Docker Images (Simplest)
```bash
# Use official images from Docker Hub
runpod/worker-comfyui:<version>-base
runpod/worker-comfyui:<version>-sdxl
runpod/worker-comfyui:<version>-sd3
runpod/worker-comfyui:<version>-flux1-schnell
runpod/worker-comfyui:<version>-flux1-dev
```

**Pros**:
- No build required
- Tested and verified
- Quick deployment

**Cons**:
- Fixed model selection
- Limited customization
- Large image sizes (15-30 GB)

#### 10.1.2 Custom Dockerfile (Most Flexible)
```dockerfile
FROM runpod/worker-comfyui:5.1.0-base

# Install custom nodes
RUN comfy-node-install comfyui-kjnodes comfyui-ic-light

# Download custom models
RUN comfy model download \
  --url https://huggingface.co/.../model.safetensors \
  --relative-path models/checkpoints/

# Copy workflow files
COPY input/ /comfyui/input/
COPY workflows/ /comfyui/user/default/workflows/
```

**Pros**:
- Full control over nodes and models
- Optimized for specific use case
- Can prebake workflow files

**Cons**:
- Requires build infrastructure
- Longer deployment time
- Need to rebuild for updates

#### 10.1.3 Network Volumes (Most Scalable)
```
Create Network Volume:
  1. RunPod Console → Storage → Create Network Volume
  2. Upload models to /models/ directory structure

Create Template:
  1. Select base image
  2. Attach network volume
  3. No code changes needed

Models automatically discovered via extra_model_paths.yaml
```

**Pros**:
- Separate model management from code
- Share models across multiple endpoints
- Update models without rebuilding
- Smaller Docker images

**Cons**:
- Additional storage cost
- Network I/O overhead
- Requires proper directory structure

### 10.2 RunPod Platform Architecture

#### 10.2.1 Template Configuration
```json
{
  "name": "ComfyUI Worker - SDXL",
  "dockerImage": "runpod/worker-comfyui:5.3.0-sdxl",
  "containerDiskInGb": 20,
  "volumeInGb": 0,
  "volumeMountPath": "/runpod-volume",
  "env": [
    {"key": "COMFY_LOG_LEVEL", "value": "INFO"},
    {"key": "BUCKET_ENDPOINT_URL", "value": "https://..."},
    {"key": "BUCKET_ACCESS_KEY_ID", "value": "..."},
    {"key": "BUCKET_SECRET_ACCESS_KEY", "value": "..."}
  ]
}
```

#### 10.2.2 Endpoint Configuration
```json
{
  "name": "My SDXL Endpoint",
  "gpuType": "NVIDIA RTX A5000",
  "gpuCount": 1,
  "workers": {
    "active": 1,
    "max": 10
  },
  "idleTimeout": 5,
  "flashBoot": true,
  "networkVolumeId": "volume-id"
}
```

**Parameters**:
- **GPU Type**: Determines VRAM, compute capability
- **Workers Active**: Minimum workers always running
- **Workers Max**: Maximum concurrent workers
- **Idle Timeout**: Minutes before scaling down
- **Flash Boot**: Fast cold start (recommended)

### 10.3 Container Lifecycle

#### 10.3.1 Startup Sequence
```
1. Container starts
   ↓
2. Run src/start.sh
   ↓
3. Load libtcmalloc
   ↓
4. Set ComfyUI-Manager to offline mode
   ↓
5. Start ComfyUI server (background)
   ↓
6. Wait for ComfyUI to be ready
   ↓
7. If SERVE_API_LOCALLY:
     Start local API simulation
   Else:
     Start RunPod handler listener
   ↓
8. Ready to process jobs
```

#### 10.3.2 Job Processing Lifecycle
```
1. Worker receives job from RunPod
   ↓
2. handler(job) called
   ↓
3. Process workflow (see Request Flow)
   ↓
4. Return result to RunPod
   ↓
5. If REFRESH_WORKER=true:
     Stop worker (clean state)
   Else:
     Wait for next job
```

### 10.4 Scaling Behavior

#### 10.4.1 Auto-scaling Rules
```
If queue_length > 0 AND active_workers < max_workers:
  Spin up new worker

If worker_idle_time > idle_timeout AND active_workers > min_workers:
  Scale down worker
```

#### 10.4.2 Cold Start Optimization
**Flash Boot** (recommended):
- Pre-loads Docker layers
- Caches model files
- Reduces startup from 60s to 5-10s

**Without Flash Boot**:
- Full container initialization
- Model loading from scratch
- 60-120s startup time

### 10.5 CI/CD Pipeline

#### 10.5.1 GitHub Actions Workflows

**release.yml** (Production Releases):
```yaml
trigger: push to main (with conventional commit)
steps:
  1. Semantic release (version bump)
  2. Docker bake (build all variants)
  3. Push to Docker Hub with tags:
     - latest
     - <version> (e.g., 5.3.0)
     - <version>-<variant> (e.g., 5.3.0-sdxl)
  4. Update Docker Hub description
  5. Create GitHub release
```

**dev.yml** (Development Builds):
```yaml
trigger: push to main (any commit)
steps:
  1. Docker bake (build all variants)
  2. Push to Docker Hub with :dev tag
  3. No versioning
```

#### 10.5.2 Build Matrix
```yaml
variants:
  - base
  - sdxl
  - sd3
  - flux1-schnell
  - flux1-dev
  - flux1-dev-fp8
  - z-image-turbo
  - base-cuda12.8.1

Build time: ~20-40 minutes per variant
Total CI time: ~2-4 hours (parallel builds)
```

---

## 11. Performance & Scalability

### 11.1 Performance Characteristics

#### 11.1.1 Request Timing Breakdown
```
Total Request Time = t_queue + t_startup + t_execution + t_output

t_queue:     0-60s     (depends on worker availability)
t_startup:   50ms-25s  (ComfyUI health check)
t_execution: 2-60s+    (model-dependent)
t_output:    0.2-5s    (image transfer + encoding/upload)
```

#### 11.1.2 GPU Memory Requirements

| Model Type | VRAM Required | Recommended GPU | Max Batch Size |
|------------|---------------|-----------------|----------------|
| SD 1.5 | 4 GB | RTX 3060 | 4 |
| SDXL | 8 GB | RTX A5000 | 2 |
| SD3 Medium | 5 GB | RTX 4070 | 2 |
| FLUX.1 schnell | 24 GB | RTX A6000 | 1 |
| FLUX.1 dev | 24 GB | RTX A6000 | 1 |

#### 11.1.3 Throughput Optimization

**Single Worker Throughput**:
```
Throughput = 1 / (t_execution + t_output)
Example: 5s execution + 0.5s output = ~0.18 images/second
```

**Multi-Worker Throughput**:
```
Total Throughput = Throughput_per_worker × Number_of_workers
Example: 0.18 img/s × 10 workers = 1.8 images/second
```

### 11.2 Scalability Patterns

#### 11.2.1 Horizontal Scaling
- **Method**: Increase max workers in endpoint configuration
- **Limits**: Platform account limits, GPU availability
- **Cost**: Linear scaling (pay per worker per second)
- **Use Case**: Handle traffic spikes, high concurrency

#### 11.2.2 Vertical Scaling
- **Method**: Use more powerful GPUs
- **Limits**: GPU tier availability, VRAM constraints
- **Cost**: Non-linear (high-end GPUs more expensive per FLOP)
- **Use Case**: Reduce per-image latency, enable larger models

#### 11.2.3 Model Optimization
- **Quantization**: FP16 → FP8 → INT8 (trade quality for speed)
- **LoRA**: Fine-tuned adapters instead of full models
- **Distilled Models**: Smaller models trained to mimic larger ones
- **Example**: FLUX.1 dev-fp8 uses half the VRAM of full precision

### 11.3 Resource Management

#### 11.3.1 Memory Management
```bash
# libtcmalloc loaded in start.sh for better memory allocation
export LD_PRELOAD=/usr/lib/$(uname -m)-linux-gnu/libtcmalloc.so.4
```

**Benefits**:
- Reduced memory fragmentation
- Faster allocation/deallocation
- Lower peak memory usage

#### 11.3.2 Container Resource Limits
```
CPU: 8-16 cores (recommended)
RAM: 16-32 GB (depends on model)
GPU: 1 (multi-GPU not currently supported)
Container Disk: 10-30 GB (depends on model size)
```

#### 11.3.3 Network Volume Performance
```
Read Performance: 100-500 MB/s
Latency: 5-20ms
Impact: 1-3s added to first model load
Mitigation: Flash Boot caches frequently used models
```

### 11.4 Bottleneck Analysis

#### 11.4.1 Common Bottlenecks

| Bottleneck | Symptom | Solution |
|------------|---------|----------|
| GPU compute | Long execution times | Use faster GPU, optimize workflow |
| VRAM | CUDA OOM errors | Use smaller model, reduce batch size |
| Network I/O | Slow model loading | Use Flash Boot, prebake models in image |
| Base64 encoding | High CPU usage | Enable S3 upload mode |
| Worker startup | High queue times | Increase min workers, enable Flash Boot |

#### 11.4.2 Monitoring Metrics
```python
# RunPod provides these metrics in dashboard:
- Queue length
- Active workers
- Average execution time
- Success rate
- Error rate
- Worker utilization
```

---

## 12. Security Considerations

### 12.1 Container Security

#### 12.1.1 ComfyUI-Manager Offline Mode
**Why**: ComfyUI-Manager can install arbitrary Python packages from internet.

**Mitigation**:
```bash
# Set in start.sh
/scripts/comfy-manager-set-mode.sh offline
```

**Effect**:
- Prevents runtime package installation
- Blocks internet access for node installation
- All dependencies must be in Docker image

#### 12.1.2 Network Isolation
```
ComfyUI listens on 127.0.0.1:8188 (localhost only)
Not exposed to public internet
Only handler.py can access
```

### 12.2 API Security

#### 12.2.1 Authentication
```http
Authorization: Bearer <runpod_api_key>
```

**Key Management**:
- Generated in RunPod console
- Scoped to user account
- Can be rotated at any time

#### 12.2.2 Input Validation
```python
# handler.py performs strict validation:
- Workflow must be dict
- Images must have 'name' and 'image'
- Base64 decoding with error handling
- JSON structure validation
```

**Prevents**:
- Code injection
- Path traversal
- Malformed data processing

### 12.3 Data Security

#### 12.3.1 Temporary File Handling
```python
# Images written to temp files for S3 upload
with tempfile.NamedTemporaryFile(delete=False, suffix=".png") as temp_file:
    temp_file.write(image_data)
    temp_path = temp_file.name

try:
    url = rp_upload.upload_image(job_id, temp_path)
finally:
    os.remove(temp_path)  # Always cleanup
```

#### 12.3.2 S3 Security
```
Bucket Policy: Private by default
Access: IAM credentials (env vars)
Path Structure: /<job_id>/<filename> (prevents collisions)
Encryption: Supports SSE-S3, SSE-KMS
```

#### 12.3.3 Comfy.org API Key Handling
```python
# Per-request key overrides global key
comfy_org_api_key = (
    input.get("comfy_org_api_key") or
    os.environ.get("COMFY_ORG_API_KEY")
)

# Not logged or exposed in responses
# Injected into ComfyUI payload as api_key_comfy_org
```

### 12.4 Workflow Security

#### 12.4.1 Untrusted Workflow Execution
**Risk**: Users can submit arbitrary ComfyUI workflows.

**Mitigations**:
1. **Sandboxed Environment**: Container isolation
2. **No Shell Execution**: ComfyUI nodes don't execute shell commands
3. **Model Whitelisting**: Only available models can be loaded
4. **Resource Limits**: GPU memory, execution time limits
5. **Offline Mode**: No internet access for dynamic downloads

#### 12.4.2 Malicious Node Protection
**Risk**: Custom nodes could contain malicious code.

**Mitigations**:
1. **Pre-installed Only**: Nodes baked into Docker image
2. **Reviewed Nodes**: Only install trusted community nodes
3. **No Runtime Installation**: ComfyUI-Manager in offline mode

### 12.5 Secrets Management

#### 12.5.1 Environment Variables
```bash
# Sensitive values passed as env vars
BUCKET_ACCESS_KEY_ID=xxx
BUCKET_SECRET_ACCESS_KEY=xxx
COMFY_ORG_API_KEY=xxx
```

**Best Practices**:
- Set in RunPod template (not in Dockerfile)
- Use RunPod's secrets management
- Rotate keys periodically

#### 12.5.2 HuggingFace Access Tokens
```dockerfile
ARG HUGGINGFACE_ACCESS_TOKEN
# Used only during build for gated models
# Not persisted in final image
```

### 12.6 Network Security

#### 12.6.1 Egress Control
```
Allowed:
- HuggingFace (model downloads during build)
- S3 endpoint (image uploads)
- Comfy.org API (if key provided)

Blocked:
- Arbitrary internet access (offline mode)
- SSH, FTP, etc.
```

#### 12.6.2 TLS/HTTPS
```
RunPod API: HTTPS only
S3 Upload: HTTPS (configurable endpoint)
Internal ComfyUI: HTTP (localhost only, no exposure)
```

---

## Appendix

### A. Network Volume Directory Structure
```
/runpod-volume/
└── models/
    ├── checkpoints/          # .safetensors, .ckpt, .pt, .pth, .bin
    │   ├── sd_xl_base_1.0.safetensors
    │   └── flux1-dev.safetensors
    ├── loras/                # .safetensors, .pt
    ├── vae/                  # .safetensors, .pt, .bin
    ├── clip/                 # .safetensors, .pt, .bin
    ├── clip_vision/          # Model files
    ├── controlnet/           # .safetensors, .pt, .pth, .bin
    ├── embeddings/           # .safetensors, .pt, .bin
    ├── upscale_models/       # .safetensors, .pt, .pth
    ├── unet/                 # .safetensors, .pt, .bin
    └── configs/              # .yaml, .json
```

### B. Common Error Messages

| Error | Cause | Fix |
|-------|-------|-----|
| "ComfyUI server not reachable" | Server failed to start | Check logs, increase container disk |
| "CUDA out of memory" | Insufficient VRAM | Use smaller model, reduce batch size |
| "Model not found" | Model not in image/volume | Add model or fix path |
| "Required input not provided" | Workflow validation failure | Check node connections |
| "WebSocket reconnection failed" | ComfyUI crashed | Check error logs, increase resources |
| "S3 upload failed" | Network/auth issue | Verify credentials and endpoint |

### C. Reference Links

- **Official Repository**: https://github.com/runpod-workers/worker-comfyui
- **Docker Hub**: https://hub.docker.com/r/runpod/worker-comfyui
- **RunPod Documentation**: https://docs.runpod.io/serverless/overview
- **ComfyUI**: https://github.com/comfyanonymous/ComfyUI
- **RunPod Console**: https://www.runpod.io/console

### D. Version History

| Version | Branch | Date | Key Changes |
|---------|--------|------|-------------|
| 5.3.0 | main | 2025-01 | WebSocket monitoring, multi-image support, FLUX.1 support |
| 5.3.0-ztex | FluidStudio-Qwen | 2026-01 | **ztex model type** (network volume mandatory), test suite removed (commits: `3349260`, `aeb6bdf`), build simplification (commit: `559eb7a`), directory structure changes (commit: `1ff5df7`) |

**FluidStudio-Qwen Branch Changelog:**
- **2026-01-31**: Documentation update (comprehensive FluidStudio-Qwen guide added)
- **Commit 1ff5df7**: Added `models/loras/` directory, explicit `unet/` configuration
- **Commit 3349260**: Removed `tests/` directory
- **Commit aeb6bdf**: Removed `test_resources/` and `.runpod/tests.json`
- **Commit 25412d5**: Build trigger update
- **Commit 559eb7a**: Removed all model downloads from Dockerfile (ztex optimization)
- **Base**: worker-comfyui v5.3.0 (upstream)

---

**Document Version**: 1.0
**Last Updated**: 2026-01-31
**Branch**: FluidStudio-Qwen
**Maintainer**: worker-comfyui-FluidStudio project
