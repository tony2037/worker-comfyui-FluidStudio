# Architecture Overview

This document provides a high-level overview of the worker-comfyui-FluidStudio architecture. For detailed technical specifications, see [spec.md](spec.md).

## System Components

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
│  │         ▼                 │  - Output retrieval      │   │ │
│  │  ┌──────────────┐         └──────────┬───────────────┘   │ │
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

## Key Components

### 1. Handler (handler.py)
The main orchestration layer that:
- Receives jobs from RunPod platform
- Validates input (workflow, images)
- Coordinates with ComfyUI server
- Monitors execution via WebSocket
- Returns results (base64 or S3 URLs)

**File**: `handler.py` (836 lines)

**Key Functions**:
- `handler()` - Main entry point (lines 507-831)
- `validate_input()` - Input validation
- `upload_images()` - Base64 image upload
- `queue_workflow()` - Workflow submission
- `_attempt_websocket_reconnect()` - Intelligent reconnection

**FluidStudio-Qwen Notes**: No changes to handler.py in this branch.

### 2. ComfyUI Server
AI workflow execution engine:
- Runs locally on `127.0.0.1:8188`
- Loads and executes workflows
- Manages GPU resources
- Generates images
- Provides HTTP/WebSocket APIs

**FluidStudio-Qwen Notes**:
- Model discovery now includes `/runpod-volume/models/loras/` and `/runpod-volume/models/unet/`
- For ztex variant, ALL models must be on network volume (nothing baked into image)

### 3. Network Volume (Mandatory for ztex)
Persistent storage for models:
- Mounted at `/runpod-volume`
- Shared across worker instances
- Configured via `extra_model_paths.yaml`
- Reduces Docker image size

**FluidStudio-Qwen Changes**:
- ⚠️ **MANDATORY for ztex variant** (not optional)
- Added `loras/` directory configuration
- Added `unet/` directory for FLUX/SD3 models
- No fallback to baked models (ztex has none)

**Required structure for ztex**:
```
/runpod-volume/
└── models/
    ├── checkpoints/    # Required if using checkpoint-based models
    ├── loras/          # Explicitly configured in this branch
    ├── vae/            # Optional
    ├── unet/           # For FLUX, SD3 (explicitly configured)
    ├── clip/           # For FLUX, SD3
    └── ... (other model types)
```

### 4. S3 Integration (Optional)
Image upload to AWS S3:
- Configured via environment variables
- Uploads generated images
- Returns presigned URLs
- Alternative to base64 encoding

## Request Flow

```
1. Client Request
   POST /runsync or /run
   ↓
2. RunPod Platform
   Routes to worker, assigns job_id
   ↓
3. Handler: Input Validation
   Validates workflow and images
   ↓
4. Handler: Server Health Check
   Ensures ComfyUI is ready
   ↓
5. Handler: Image Upload (if any)
   Uploads base64 images to ComfyUI
   ↓
6. Handler: WebSocket Setup
   Establishes real-time connection
   ↓
7. Handler: Queue Workflow
   Submits workflow for execution
   ↓
8. Monitor Execution
   Tracks progress via WebSocket
   Handles errors and reconnections
   ↓
9. Fetch Results
   Gets execution history
   ↓
10. Process Output Images
    Downloads images from ComfyUI
    ↓
11. Deliver Results
    Base64 encode OR upload to S3
    ↓
12. Return Response
    Images array with metadata
```

## Communication Patterns

### HTTP APIs
- **Client → RunPod**: HTTPS REST API
- **Handler → ComfyUI**: HTTP (localhost only)
  - `GET /` - Health check
  - `POST /upload/image` - Image upload
  - `POST /prompt` - Queue workflow
  - `GET /history/{prompt_id}` - Fetch results
  - `GET /view` - Download images

### WebSocket
- **Handler ↔ ComfyUI**: Real-time monitoring
  - `ws://127.0.0.1:8188/ws?clientId={uuid}`
  - Message types: status, executing, execution_error
  - Automatic reconnection on disconnect

## Data Flow

### Input Data
```json
{
  "input": {
    "workflow": {...},           // ComfyUI node graph
    "images": [{                 // Optional input images
      "name": "...",
      "image": "base64..."
    }],
    "comfy_org_api_key": "..."   // Optional API key
  }
}
```

### Output Data
```json
{
  "images": [
    {
      "filename": "ComfyUI_00001_.png",
      "type": "base64" | "s3_url",
      "data": "..." // Base64 string or S3 URL
    }
  ],
  "errors": ["..."] // Optional warnings
}
```

## Deployment Architecture

### Docker Image Variants
- **base**: Clean ComfyUI, no models (~5 GB)
- **sdxl**: SDXL models included (~15 GB)
- **sd3**: SD3 Medium (~20 GB)
- **flux1-schnell**: FLUX.1 schnell (~30 GB)
- **flux1-dev**: FLUX.1 dev (~30 GB)
- **flux1-dev-fp8**: FLUX.1 dev FP8 (~30 GB)
- **ztex**: FluidStudio custom - no models, network volume mandatory (~5-7 GB, current branch)

### Container Lifecycle
```
1. Container Start
   ↓
2. Load libtcmalloc (memory optimization)
   ↓
3. Configure ComfyUI-Manager (offline mode)
   ↓
4. [ztex] Verify network volume mounted (MANDATORY)
   ↓
5. Start ComfyUI Server (background)
   ↓
6. Start RunPod Handler
   ↓
7. Process Jobs
   ↓
8. Optional: Refresh Worker (clean state)
```

**FluidStudio-Qwen ztex variant**: Step 4 is critical. If network volume not mounted, ComfyUI will start but have no models, causing all jobs to fail with "model not found" errors.

### Scaling Model
- **Horizontal**: Multiple worker instances
- **Auto-scaling**: Based on queue length
- **GPU per Worker**: 1 GPU per worker
- **Stateless**: Each job can run on any worker

## Key Design Decisions

### 1. WebSocket for Monitoring
**Why**: Real-time progress tracking, immediate error detection
**Replaces**: Polling-based approach in v4.x
**Benefits**: Lower latency, reduced API calls, better UX

### 2. Intelligent Reconnection
**Why**: WebSocket can drop during long executions
**How**: Check HTTP health before reconnect attempts
**Benefits**: Distinguish between connection issues and server crashes

### 3. Base64 vs S3 Output
**Why**: RunPod has request size limits (10-20 MB)
**Base64**: Simple, no setup required, works for small images
**S3**: Required for large/multiple images, more scalable

### 4. Network Volumes
**Why**: Models are large (5-30 GB), slow to bake into images
**How**: Persistent storage mounted at runtime
**Benefits**: Faster deploys, shared resources, dynamic updates

### 5. Offline ComfyUI-Manager
**Why**: Security - prevent arbitrary code execution
**How**: Set to offline mode in start.sh
**Trade-off**: All nodes must be pre-installed in image

### 6. Localhost-only ComfyUI
**Why**: Security - no public access to ComfyUI
**How**: ComfyUI listens on 127.0.0.1:8188
**Benefits**: Handler is only interface to outside world

## FluidStudio-Qwen Branch Specifics

### Overview
The `FluidStudio-Qwen` branch represents a **production-optimized fork** of worker-comfyui v5.3.0, designed for scenarios where:
- Models change frequently (don't want to rebuild images)
- Multiple endpoints share the same models (network volume efficiency)
- Fast build/deployment cycles are critical
- Docker image size matters (cost/storage)

### The ztex Model Type

**What it is:**
- A Docker image variant with **zero pre-downloaded models**
- Requires network volume for all models (mandatory, not optional)
- Named after FluidStudio internal identifier

**Build configuration** (Dockerfile lines 108-111):
```dockerfile
RUN if [ "$MODEL_TYPE" = "ztex" ]; then \
      echo "ztex"; \
      echo "done"; \
    fi
```

**Contrast with other variants**:
```dockerfile
# sdxl variant:
RUN wget https://huggingface.co/.../sd_xl_base_1.0.safetensors ...
RUN wget https://huggingface.co/.../sd_xl_vae.safetensors ...
# (many more downloads, 15-30 GB)

# ztex variant:
# (nothing - just echo statements)
```

**Benefits:**
- Build time: 5-10 min (vs 30-60 min for model variants)
- Image size: ~5-7 GB (vs 15-30 GB)
- No HuggingFace token required during build
- Models updated independently of image

**Trade-offs:**
- Network volume mandatory (not optional)
- Slightly slower first request (model loading from network)
- More deployment steps (need to pre-upload models)

### Test Suite Removal

**What was removed:**
- `tests/` directory (unit and integration tests)
- `test_resources/` directory (test fixtures, sample workflows)
- `.runpod/tests.json` (RunPod test configuration)

**Commits:**
- `3349260` - "remove tests"
- `aeb6bdf` - "Remove test"

**Rationale:**
This branch is focused on **production deployment**, not active development:
- Core functionality tested in upstream repository
- Production deployments require end-to-end testing regardless
- Reduces maintenance burden
- Simplifies codebase for deployment-focused use

**Impact on users:**
- Developers must test changes manually
- See [development.md](development.md) for recommended testing workflow
- Upstream tests remain available for reference

### Directory Structure Changes

**Additions** (commit `1ff5df7`):
```yaml
# src/extra_model_paths.yaml
runpod_worker_comfy:
  base_path: /runpod-volume
  loras: models/loras/          # Explicitly added
  unet: models/unet/            # Explicitly added (for FLUX/SD3)
  # ... other paths unchanged
```

**Purpose:**
- Ensure ComfyUI discovers LoRA models on network volume
- Support FLUX.1 and SD3 (which use `unet/` instead of `checkpoints/`)
- Explicit configuration prevents model discovery issues

**Required network volume structure for ztex:**
```
/runpod-volume/
└── models/
    ├── checkpoints/    # Traditional SD models
    ├── loras/          # LoRA models (explicitly configured)
    ├── vae/            # VAE models
    ├── unet/           # FLUX, SD3 models (explicitly configured)
    ├── clip/           # CLIP text encoders
    └── ... (other types as needed)
```

### Build Simplification

**Commit `559eb7a`**: "Fix: Do not download models"

**What changed:**
- All `wget` commands removed from Dockerfile
- No model downloads during build
- No `HUGGINGFACE_ACCESS_TOKEN` handling

**Before (other variants):**
```dockerfile
ARG HUGGINGFACE_ACCESS_TOKEN
RUN wget --header="Authorization: Bearer ${HUGGINGFACE_ACCESS_TOKEN}" \
  https://huggingface.co/.../model.safetensors ...
# (many more wget commands)
```

**After (ztex):**
```dockerfile
# (no downloads)
```

**Benefits:**
- Faster CI/CD (no download time)
- No secret management in build process
- More reliable builds (no download failures)
- Cleaner build logs

### Deployment Considerations

**For ztex deployments, you MUST:**
1. ✅ Create RunPod network volume
2. ✅ Upload models to `/runpod-volume/models/<type>/`
3. ✅ Attach network volume to endpoint (in Advanced settings)
4. ✅ Verify models detected (use `NETWORK_VOLUME_DEBUG=true`)

**Common mistake:**
Creating endpoint without attaching network volume. ComfyUI will start successfully but have no models, causing all jobs to fail with "model not found."

**Debugging:**
```bash
# Environment variable in endpoint settings:
NETWORK_VOLUME_DEBUG=true

# Logs will show:
# ✓ MOUNTED: /runpod-volume
# ✓ FOUND: /runpod-volume/models
# ✓ Models found: checkpoints/my-model.safetensors
```

### When to Use ztex vs Other Variants

**Use ztex when:**
- Models change frequently
- Sharing models across multiple endpoints
- Fast deployment cycles needed
- Docker image size/cost matters
- You already have model management infrastructure

**Use model-included variants (sdxl, flux1-dev, etc.) when:**
- Models are static/unchanging
- Single-purpose endpoint (one model only)
- Want simplest deployment (no network volume setup)
- Cold start latency critical (models pre-loaded)

**Hybrid approach:**
You can mix both! Use ztex image but also create custom variant with frequently-used models baked in. ComfyUI searches both locations.

### Migration from Other Variants

**From model-included variant to ztex:**
1. Extract models from current image
2. Upload to network volume
3. Change template to use ztex image
4. Attach network volume to endpoint
5. Test with `NETWORK_VOLUME_DEBUG=true`

**From base variant:**
Already similar architecture. Main differences:
- ztex has explicit `loras/` and `unet/` configuration
- ztex is production-focused (tests removed)

## Performance Characteristics

### Typical Request Timing
```
Queue Wait:      0-60s     (depends on worker availability)
Startup Check:   50ms-25s  (ComfyUI health check)
Image Upload:    100ms-2s  (per image)
Workflow Queue:  50-200ms  (validation + submission)
Execution:       2-60s+    (model-dependent)
Image Download:  100ms-1s  (per image)
S3 Upload:       200ms-3s  (per image)
Base64 Encode:   50-200ms  (per image)
```

### GPU Memory Requirements
| Model | VRAM | Recommended GPU |
|-------|------|-----------------|
| SD 1.5 | 4 GB | RTX 3060 |
| SDXL | 8 GB | RTX A5000 |
| SD3 Medium | 5 GB | RTX 4070 |
| FLUX.1 | 24 GB | RTX A6000 |

### Throughput
- **Single Worker**: ~0.1-0.5 images/second (depends on model)
- **Multi-Worker**: Linear scaling (10 workers = 10x throughput)
- **Bottlenecks**: GPU compute, VRAM, network I/O

## Error Handling Strategy

### Validation Errors (Client)
- Input structure validation
- Workflow node validation
- Missing model detection
- Detailed error messages with available options

### Runtime Errors (Server)
- WebSocket reconnection (up to 5 attempts)
- Partial failure handling (continue with what works)
- Graceful degradation (S3 failure → base64 fallback)
- Comprehensive logging

### Error Response Format
```json
{
  "error": "Error description",
  "details": [
    "Node 10 (KSampler): CUDA out of memory",
    "Available VRAM: 12 GB",
    "Required: 16 GB"
  ]
}
```

## Security Considerations

### Container Security
- ComfyUI-Manager in offline mode (no arbitrary package installation)
- ComfyUI listens on localhost only (not exposed)
- Sandboxed execution environment

### API Security
- RunPod API key authentication
- Input validation and sanitization
- No shell command execution

### Data Security
- Temporary file cleanup
- S3 with IAM credentials (not hardcoded)
- API keys not logged or exposed

### Network Security
- HTTPS for external APIs
- Limited egress (S3, HuggingFace, Comfy.org only)
- No arbitrary internet access

## Configuration

### Environment Variables
| Variable | Purpose | Default |
|----------|---------|---------|
| `REFRESH_WORKER` | Stop after each job | `false` |
| `COMFY_LOG_LEVEL` | Logging verbosity | `DEBUG` |
| `NETWORK_VOLUME_DEBUG` | Volume diagnostics | `false` |
| `WEBSOCKET_RECONNECT_ATTEMPTS` | Max reconnects | `5` |
| `WEBSOCKET_RECONNECT_DELAY_S` | Reconnect delay | `3` |
| `BUCKET_ENDPOINT_URL` | S3 endpoint (enables S3) | – |
| `BUCKET_ACCESS_KEY_ID` | AWS access key | – |
| `BUCKET_SECRET_ACCESS_KEY` | AWS secret key | – |
| `COMFY_ORG_API_KEY` | Comfy.org API key | – |

### Hardcoded Constants
```python
COMFY_HOST = "127.0.0.1:8188"
COMFY_API_AVAILABLE_INTERVAL_MS = 50
COMFY_API_AVAILABLE_MAX_RETRIES = 500
```

## File Structure

```
/Users/tonyliu/workspace/worker-comfyui-FluidStudio/
├── handler.py                    # Main orchestrator (836 lines)
├── src/
│   ├── network_volume.py         # Volume diagnostics (153 lines)
│   ├── start.sh                  # Container startup (25 lines)
│   ├── extra_model_paths.yaml    # Model path config (12 lines)
│   └── restore_snapshot.sh       # Snapshot restoration
├── scripts/
│   ├── comfy-node-install.sh     # Safe node installation (44 lines)
│   └── comfy-manager-set-mode.sh # ComfyUI-Manager config (32 lines)
├── Dockerfile                     # Multi-stage build (117 lines)
├── docker-bake.hcl               # Build variants definition
├── docker-compose.yml            # Local development setup
├── requirements.txt              # Python dependencies
└── docs/
    ├── README.md                 # Documentation index
    ├── spec.md                   # Detailed technical spec
    ├── architecture.md           # This file
    ├── deployment.md             # Deployment guide
    ├── configuration.md          # Configuration reference
    ├── customization.md          # Customization guide
    ├── development.md            # Development guide
    ├── network-volumes.md        # Network volume guide
    ├── ci-cd.md                  # CI/CD documentation
    └── fluidstudio-qwen.md       # FluidStudio-Qwen branch guide (NEW)
```

**FluidStudio-Qwen Branch Changes:**
- ❌ Removed: `tests/` directory (unit and integration tests)
- ❌ Removed: `test_resources/` directory (test fixtures)
- ❌ Removed: `.runpod/tests.json` (test configuration)
- ✅ Added: `docs/fluidstudio-qwen.md` (comprehensive branch documentation)

## Related Documentation

- **[spec.md](spec.md)**: Complete technical specification
- **[deployment.md](deployment.md)**: Deployment instructions
- **[configuration.md](configuration.md)**: Environment variables reference
- **[customization.md](customization.md)**: Adding models and nodes
- **[development.md](development.md)**: Local development setup
- **[network-volumes.md](network-volumes.md)**: Network volume guide
- **[ci-cd.md](ci-cd.md)**: CI/CD pipeline details

## Quick References

### Starting the Worker Locally
```bash
docker-compose up --build
# Access API: http://localhost:8000
# Access ComfyUI: http://localhost:8188
```

### Building Custom Image
```bash
docker build -t my-comfyui-worker \
  --build-arg MODEL_TYPE=sdxl \
  .
```

### Using with RunPod
```bash
# Create template with image
runpod/worker-comfyui:5.3.0-sdxl

# Deploy endpoint
# Configure GPU type, workers, environment variables
```

### Submitting a Job
```bash
curl -X POST \
  -H "Authorization: Bearer <api_key>" \
  -H "Content-Type: application/json" \
  -d '{"input":{"workflow":{...}}}' \
  https://api.runpod.ai/v2/<endpoint_id>/runsync
```

---

For more detailed information, see [spec.md](spec.md).
