# FluidStudio-Qwen Branch Guide

This guide provides comprehensive documentation for the `FluidStudio-Qwen` branch of `worker-comfyui-FluidStudio`, explaining its purpose, key differences from upstream, architecture, deployment, and troubleshooting.

## Table of Contents

- [Overview](#overview)
- [Key Differences from Upstream](#key-differences-from-upstream)
- [Architecture](#architecture)
- [Deployment Guide](#deployment-guide)
- [Troubleshooting](#troubleshooting)
- [Advanced Configuration](#advanced-configuration)
- [Migration Guide](#migration-guide)
- [FAQ](#faq)

## Overview

### Purpose

The `FluidStudio-Qwen` branch is a customized version of the RunPod worker-comfyui project, optimized for production deployment scenarios where models are managed via network volumes rather than baked into Docker images.

### Status

- **Current Version**: Based on worker-comfyui v5.3.0
- **Branch**: `FluidStudio-Qwen`
- **Primary Use Case**: Production deployments with network volume-based model management
- **Default Model Type**: `ztex` (no pre-downloaded models)

### Key Features

✅ Fast build times (5-10 minutes vs 30-60 minutes for model-included variants)
✅ Network volume-based model management (mandatory for ztex)
✅ Simplified Dockerfile (no model downloads)
✅ Production-focused (test suite removed)
✅ Compatible with all upstream features (WebSocket, S3, ComfyUI-Manager)

## Key Differences from Upstream

### 1. The `ztex` Model Type

The `ztex` model type is the primary innovation of this branch.

**What it is:**
- A Docker image variant with **no pre-downloaded models**
- Designed to work exclusively with network volumes
- Named after the FluidStudio customization (ztex = FluidStudio internal identifier)

**How it differs from other variants:**
```dockerfile
# Other variants (sdxl, sd3, flux1-dev, etc.):
RUN wget https://huggingface.co/... -O models/checkpoints/model.safetensors
RUN wget https://huggingface.co/... -O models/vae/vae.safetensors
# ... many more downloads

# ztex variant (lines 108-111 in Dockerfile):
RUN if [ "$MODEL_TYPE" = "ztex" ]; then \
      echo "ztex"; \
      echo "done"; \
    fi
```

**Benefits:**
- ⚡ **Fast builds**: 5-10 minutes (no model downloads)
- 🔒 **No HuggingFace token required**: No downloads during build
- 💾 **Small base image**: ~5-7 GB (vs 15-30 GB for model-included variants)
- 🔄 **Flexible model management**: Update models without rebuilding images
- 🎯 **Production-ready**: Separates infrastructure from data

**Requirements:**
- ⚠️ **Network volume is MANDATORY** (not optional like other variants)
- Models must be pre-uploaded to `/runpod-volume/models/`
- Directory structure must follow ComfyUI conventions

### 2. Test Suite Removal

**What was removed:**
- `tests/` directory (unit and integration tests)
- `test_resources/` directory (test fixtures and sample workflows)
- `.runpod/tests.json` (RunPod test configuration)

**Commits:**
- `3349260` - "remove tests"
- `aeb6bdf` - "Remove test"

**Rationale:**
- **Production focus**: This branch is for deployment, not development
- **Upstream testing**: Core functionality tested in upstream repository
- **Manual verification**: Production deployments require end-to-end testing anyway
- **Reduced complexity**: Fewer dependencies and maintenance burden

**Impact:**
- Developers must test changes manually before deployment
- See [Testing Your Changes](#testing-your-changes) for recommended workflow
- Upstream tests can still be referenced for guidance

### 3. Directory Structure Changes

**Added directories:**
```bash
models/loras/          # LoRA models directory (created in Dockerfile:105)
```

**Updated configuration:**
```yaml
# src/extra_model_paths.yaml
runpod_worker_comfy:
  base_path: /runpod-volume
  # ... existing paths ...
  loras: models/loras/       # Explicitly configured
  unet: models/unet/         # Added for FLUX and SD3 models
```

**Commit:**
- `1ff5df7` - "Fix: change the dir structure back"

**Purpose:**
- Ensure compatibility with latest ComfyUI model organization
- Support for FLUX, SD3, and other modern architectures
- Explicit configuration prevents model discovery issues

### 4. Build Simplification

**What changed:**
- All `wget` model download commands removed from Dockerfile
- No HuggingFace token handling in build process
- Minimal downloader stage (lines 95-111)

**Commit:**
- `559eb7a` - "Fix: Do not download models"

**Benefits:**
- Faster CI/CD pipeline
- No secret management during builds
- Reduced build failure surface (no download timeouts/errors)
- Cleaner build logs

### 5. Accurate Technical Details

**Line counts** (as of current branch):
- `handler.py`: 836 lines (not 837)
- `network_volume.py`: 153 lines
- `start.sh`: 25 lines

These counts are used in documentation for accuracy and traceability.

## Architecture

### Model Loading Flow (ztex)

```
1. Container Startup
   ↓
2. Check /runpod-volume mount
   ├─ If not mounted → ERROR (ztex requires network volume)
   └─ If mounted → Continue
   ↓
3. Load extra_model_paths.yaml
   Configures ComfyUI to look at /runpod-volume/models/
   ↓
4. ComfyUI Server Starts
   Scans for models in:
   - /comfyui/models/ (empty for ztex)
   - /runpod-volume/models/ (network volume)
   ↓
5. Ready to Process Jobs
   Uses models from network volume
```

### Network Volume Structure (Required for ztex)

```
/runpod-volume/
└── models/
    ├── checkpoints/       # Stable Diffusion checkpoints
    │   └── my-model.safetensors
    ├── loras/             # LoRA models (required directory)
    │   └── style-lora.safetensors
    ├── vae/               # VAE models
    │   └── vae-ft-mse.safetensors
    ├── unet/              # UNet models (FLUX, SD3)
    │   └── flux1-dev.safetensors
    ├── clip/              # CLIP text encoders
    ├── clip_vision/       # CLIP vision models
    ├── controlnet/        # ControlNet models
    ├── embeddings/        # Textual inversions
    ├── upscale_models/    # Upscalers (RealESRGAN, etc.)
    └── configs/           # Model configs (.yaml)
```

**Important notes:**
- The `models/` directory under `/runpod-volume/` is mandatory
- Only create subdirectories you actually need
- ComfyUI only recognizes specific file extensions (`.safetensors`, `.ckpt`, `.pt`, `.pth`, `.bin`)
- Files must be in the correct subdirectory (e.g., checkpoints in `checkpoints/`, not `loras/`)

### Debugging Network Volume

If models are not detected, enable debug mode:

```bash
# In RunPod endpoint environment variables:
NETWORK_VOLUME_DEBUG=true
```

This prints detailed diagnostics on every request:
```
======================================================================
NETWORK VOLUME DIAGNOSTICS (NETWORK_VOLUME_DEBUG=true)
======================================================================

[1] Checking extra_model_paths.yaml configuration...
    ✓ FOUND: /comfyui/extra_model_paths.yaml

[2] Checking network volume mount at /runpod-volume...
    ✓ MOUNTED: /runpod-volume

[3] Checking directory structure...
    ✓ FOUND: /runpod-volume/models

[4] Scanning model directories...

    checkpoints/:
      - my-model.safetensors (6.5 GB)

    loras/:
      - style-lora.safetensors (144.2 MB)

    unet/:
      - flux1-dev.safetensors (23.8 GB)

[5] Summary
    ✓ Models found on network volume!
======================================================================
```

See the [Troubleshooting](#troubleshooting) section for common issues.

## Deployment Guide

### Prerequisites

1. **RunPod Account** with serverless access
2. **Network Volume** with models pre-uploaded
3. **Docker Image**: `runpod/worker-comfyui:latest-ztex` (or build custom)
4. **GPU**: Choose based on your models (see [GPU Recommendations](#gpu-recommendations))

### Step 1: Prepare Network Volume

#### Create Network Volume

1. Navigate to [RunPod Network Volumes](https://www.runpod.io/console/serverless/user/storage)
2. Click `+ Network Volume`
3. Configure:
   - Name: `comfyui-models` (or your preference)
   - Size: Based on your models (50-500 GB typical)
   - Region: Same as your endpoint deployment
4. Click `Create`

#### Upload Models

**Option A: Using RunPod Web Interface**
1. Go to your network volume
2. Use the file manager to upload models
3. Ensure directory structure matches [Network Volume Structure](#network-volume-structure-required-for-ztex)

**Option B: Using S3-Compatible API**
```bash
# Configure AWS CLI with RunPod credentials
aws configure --profile runpod
# AWS Access Key ID: <from RunPod>
# AWS Secret Access Key: <from RunPod>
# Default region: us-east-1
# Default output format: json

# Upload models
aws s3 cp my-model.safetensors \
  s3://<NETWORK_VOLUME_ID>/models/checkpoints/my-model.safetensors \
  --profile runpod \
  --endpoint-url https://storage.runpod.io
```

**Option C: Using RunPod Pod**
1. Create a temporary pod with network volume attached
2. Upload models via SSH/Jupyter
3. Terminate pod when done (keeps network volume)

#### Verify Structure

```bash
# Expected structure:
/runpod-volume/
└── models/
    ├── checkpoints/
    ├── loras/
    ├── vae/
    └── unet/
```

### Step 2: Create Serverless Template

1. Navigate to [Serverless Templates](https://runpod.io/console/serverless/user/templates)
2. Click `New Template`
3. Configure:
   - **Template Name**: `worker-comfyui-ztex`
   - **Template Type**: `serverless`
   - **Container Image**: `runpod/worker-comfyui:5.3.0-ztex`
   - **Container Disk**: `10 GB` (ztex base image is ~5-7 GB)
   - **Container Registry Credentials**: Leave empty (public image)
4. (Optional) Environment Variables:
   ```bash
   COMFY_LOG_LEVEL=INFO
   # Add S3 credentials if using S3 upload:
   BUCKET_ENDPOINT_URL=https://s3.amazonaws.com
   BUCKET_ACCESS_KEY_ID=<your-key>
   BUCKET_SECRET_ACCESS_KEY=<your-secret>
   ```
5. Click `Save Template`

### Step 3: Create Serverless Endpoint

1. Navigate to [Serverless Endpoints](https://www.runpod.io/console/serverless/user/endpoints)
2. Click `New Endpoint`
3. Configure:

   **Basic Settings:**
   - **Endpoint Name**: `comfyui-ztex`
   - **Select Template**: `worker-comfyui-ztex`

   **Worker Configuration:**
   - **GPU Type**: Choose based on models (see [GPU Recommendations](#gpu-recommendations))
   - **Active Workers**: `0` (auto-scale based on demand)
   - **Max Workers**: `3` (adjust based on budget)
   - **GPUs/Worker**: `1`
   - **Idle Timeout**: `5` seconds

   **Advanced:**
   - ⚠️ **CRITICAL**: Under `Select Network Volume`, choose your `comfyui-models` volume
   - **Flash Boot**: `enabled` (recommended)

4. Click `Deploy`

### Step 4: Test Deployment

Wait for the endpoint to show as "Ready" (green status).

**Test request:**
```bash
curl -X POST https://api.runpod.ai/v2/<endpoint-id>/runsync \
  -H "Authorization: Bearer <your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "workflow": {
        "3": {
          "class_type": "KSampler",
          "inputs": {
            "seed": 42,
            "steps": 20,
            "cfg": 7.0,
            "sampler_name": "euler",
            "scheduler": "normal",
            "denoise": 1.0,
            "model": ["4", 0],
            "positive": ["6", 0],
            "negative": ["7", 0],
            "latent_image": ["5", 0]
          }
        },
        "4": {
          "class_type": "CheckpointLoaderSimple",
          "inputs": {
            "ckpt_name": "my-model.safetensors"
          }
        }
      }
    }
  }'
```

**Expected response:**
```json
{
  "id": "sync-...",
  "status": "COMPLETED",
  "output": {
    "images": [
      {
        "filename": "ComfyUI_00001_.png",
        "type": "base64",
        "data": "iVBORw0KGgoAAAANSUhEUgAA..."
      }
    ]
  }
}
```

If you see an error about missing models, see [Troubleshooting](#troubleshooting).

### GPU Recommendations

| Model Type | VRAM Required | Recommended GPU | Container Disk (ztex) |
|------------|---------------|-----------------|----------------------|
| SD 1.5 | 4 GB | RTX 3060 | 10 GB |
| SDXL | 8 GB | RTX A5000 | 10 GB |
| SD3 Medium | 5 GB | RTX 4070 | 10 GB |
| FLUX.1 Schnell | 16 GB | RTX A6000 | 10 GB |
| FLUX.1 Dev | 24 GB | RTX A6000 / H100 | 10 GB |

**Note:** Container disk for ztex is always small (~10 GB) since models are on network volume.

## Troubleshooting

### Models Not Detected

**Symptom:**
```json
{
  "error": "CheckpointLoaderSimple: model 'my-model.safetensors' not found",
  "details": [
    "Available checkpoints: []"
  ]
}
```

**Diagnosis:**
1. Enable debug mode:
   ```bash
   # In endpoint environment variables:
   NETWORK_VOLUME_DEBUG=true
   ```
2. Send a test request
3. Check worker logs for diagnostics

**Common causes:**

| Issue | Solution |
|-------|----------|
| Network volume not attached | Edit endpoint → Advanced → Select Network Volume |
| Wrong directory structure | Models must be in `/runpod-volume/models/checkpoints/`, not `/runpod-volume/checkpoints/` |
| Wrong file extension | Use `.safetensors`, `.ckpt`, `.pt`, or `.pth` (not `.txt`, `.zip`) |
| Models in wrong subdirectory | Checkpoints go in `checkpoints/`, LoRAs in `loras/`, etc. |
| Empty directories | Ensure model files actually uploaded (check file sizes) |

### WebSocket Disconnection

**Symptom:**
```json
{
  "error": "WebSocket disconnected during execution"
}
```

**Solution:**
This is handled automatically by the worker (up to 5 reconnection attempts). If persisting:

1. Check ComfyUI logs for crashes:
   ```bash
   # In worker logs, look for:
   "ComfyUI process exited"
   "CUDA out of memory"
   ```
2. If OOM errors, use a GPU with more VRAM
3. If random crashes, report issue with workflow

### S3 Upload Failures

**Symptom:**
```json
{
  "images": [...],
  "errors": ["Failed to upload to S3, falling back to base64"]
}
```

**Solution:**
1. Verify S3 credentials in environment variables:
   ```bash
   BUCKET_ENDPOINT_URL=https://s3.amazonaws.com
   BUCKET_ACCESS_KEY_ID=<valid-key>
   BUCKET_SECRET_ACCESS_KEY=<valid-secret>
   ```
2. Check IAM permissions (need `s3:PutObject`)
3. Worker falls back to base64 automatically (not a critical error)

### Image Too Large for Response

**Symptom:**
```json
{
  "error": "Response too large (>20MB)"
}
```

**Solution:**
Configure S3 upload to avoid base64 encoding large images:
```bash
# Environment variables:
BUCKET_ENDPOINT_URL=https://s3.amazonaws.com
BUCKET_ACCESS_KEY_ID=<key>
BUCKET_SECRET_ACCESS_KEY=<secret>
```

### Network Volume Mount Failed

**Symptom:**
```
[2] Checking network volume mount at /runpod-volume...
    ✗ NOT MOUNTED: /runpod-volume
```

**Solution:**
1. Edit endpoint → Advanced → Select Network Volume
2. Ensure volume is in same region as endpoint
3. Wait for workers to restart
4. Test again

### ComfyUI Server Not Starting

**Symptom:**
```
ComfyUI API not available after 25 seconds
```

**Solution:**
1. Check worker logs for Python errors
2. Verify GPU is available (`nvidia-smi`)
3. Check VRAM availability (might be OOM on startup)
4. Restart endpoint

## Advanced Configuration

### Customizing the ztex Model Type

If you want to create a custom variant that includes some models:

#### Option 1: Hybrid Approach (Some Baked, Some Network Volume)

**Use case:** Fast startup for common models, network volume for custom models.

**Dockerfile:**
```dockerfile
# After line 108, add model downloads:
RUN if [ "$MODEL_TYPE" = "ztex" ]; then \
      echo "Downloading base models..."; \
      wget https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.safetensors \
        -O models/checkpoints/v1-5-pruned-emaonly.safetensors; \
      echo "Base models included, custom models via network volume"; \
    fi
```

**Benefits:**
- Faster first startup (no model download wait)
- Still supports dynamic models via network volume
- Good for frequently-used base models

**Trade-offs:**
- Larger Docker image
- Longer build times
- Less flexible (need rebuild to change baked models)

#### Option 2: Custom Model Type

**Use case:** Create a new model type (e.g., `ztex-flux`) with specific models.

**Dockerfile:**
```dockerfile
RUN if [ "$MODEL_TYPE" = "ztex-flux" ]; then \
      echo "Downloading FLUX models..."; \
      wget --header="Authorization: Bearer ${HUGGINGFACE_ACCESS_TOKEN}" \
        https://huggingface.co/black-forest-labs/FLUX.1-dev/resolve/main/flux1-dev.safetensors \
        -O models/unet/flux1-dev.safetensors; \
      wget https://huggingface.co/comfyanonymous/flux_text_encoders/resolve/main/clip_l.safetensors \
        -O models/clip/clip_l.safetensors; \
      wget https://huggingface.co/comfyanonymous/flux_text_encoders/resolve/main/t5xxl_fp16.safetensors \
        -O models/clip/t5xxl_fp16.safetensors; \
      wget https://huggingface.co/black-forest-labs/FLUX.1-dev/resolve/main/ae.safetensors \
        -O models/vae/ae.safetensors; \
      echo "FLUX models included"; \
    fi
```

**Build:**
```bash
docker build \
  --build-arg MODEL_TYPE=ztex-flux \
  --build-arg HUGGINGFACE_ACCESS_TOKEN=<token> \
  --platform linux/amd64 \
  -t my-registry/worker-comfyui:ztex-flux \
  .
```

**Benefits:**
- Self-contained (no network volume required)
- Faster startup (no model loading from network)
- Reproducible (same models every time)

**Trade-offs:**
- Large image (30-40 GB for FLUX)
- Long build times (30-60 minutes)
- Need rebuild to update models

#### Option 3: Network Volume Only (Recommended)

**Use case:** Maximum flexibility, fastest builds.

**This is the default ztex behavior.** No changes needed.

**Benefits:**
- Fast builds (5-10 minutes)
- Update models without rebuilding
- Share models across endpoints
- Small Docker image

**Trade-offs:**
- Network volume required (mandatory)
- Slightly slower first request (model loading)

### Environment Variables

All upstream environment variables are supported:

| Variable | Purpose | Default | ztex Notes |
|----------|---------|---------|-----------|
| `REFRESH_WORKER` | Stop after each job | `false` | Same as upstream |
| `COMFY_LOG_LEVEL` | Log verbosity | `DEBUG` | Same as upstream |
| `NETWORK_VOLUME_DEBUG` | Volume diagnostics | `false` | **Use when debugging models** |
| `WEBSOCKET_RECONNECT_ATTEMPTS` | Max reconnects | `5` | Same as upstream |
| `WEBSOCKET_RECONNECT_DELAY_S` | Reconnect delay | `3` | Same as upstream |
| `BUCKET_ENDPOINT_URL` | S3 endpoint | – | Same as upstream |
| `BUCKET_ACCESS_KEY_ID` | AWS access key | – | Same as upstream |
| `BUCKET_SECRET_ACCESS_KEY` | AWS secret key | – | Same as upstream |
| `COMFY_ORG_API_KEY` | Comfy.org API key | – | Same as upstream |

### ComfyUI-Manager Configuration

ComfyUI-Manager is set to **offline mode** by default for security.

**To enable online mode** (not recommended for production):
```bash
# In endpoint environment variables:
COMFYUI_MANAGER_MODE=online
```

**Installing custom nodes:**
Use the `comfy-node-install` script in your custom Dockerfile:
```dockerfile
RUN comfy-node-install \
  https://github.com/ltdrdata/ComfyUI-Manager.git \
  https://github.com/WASasquatch/was-node-suite-comfyui.git
```

## Migration Guide

### From Upstream worker-comfyui

**If you're using a model-included variant** (sdxl, sd3, flux1-dev, etc.):

1. **Assess your models:**
   - List models currently baked into your image
   - Determine which are actively used

2. **Set up network volume:**
   - Create RunPod network volume
   - Upload models (see [Step 1: Prepare Network Volume](#step-1-prepare-network-volume))

3. **Update template:**
   - Change image from `runpod/worker-comfyui:5.3.0-sdxl` to `runpod/worker-comfyui:5.3.0-ztex`
   - Reduce container disk to `10 GB`

4. **Update endpoint:**
   - Attach network volume in Advanced settings
   - Test with existing workflows

5. **Verify:**
   - Use `NETWORK_VOLUME_DEBUG=true` to confirm models detected
   - Run test workflows
   - Monitor performance (first request may be slower due to model loading)

**If you're using the base variant with network volumes:**

Good news! You're already using the recommended architecture. Just switch to the ztex image for the benefits:
- Cleaner separation (ztex is designed for this)
- Explicit configuration (loras/, unet/ directories)
- Better documentation

### From Other Branches

**If you're on a custom branch:**

1. **Review your customizations:**
   - Custom nodes → Install via `comfy-node-install` script
   - Custom models → Upload to network volume
   - Custom code → Port to FluidStudio-Qwen branch

2. **Merge or rebase:**
   ```bash
   git checkout FluidStudio-Qwen
   git merge your-custom-branch
   # Resolve conflicts (primarily in Dockerfile)
   ```

3. **Test thoroughly:**
   - Since tests were removed, manual testing is critical
   - Use staging endpoint before production
   - Verify all workflows

### Testing Your Changes

**Recommended workflow** (since test suite removed):

1. **Local testing** with docker-compose:
   ```bash
   docker-compose up --build
   # Access ComfyUI: http://localhost:8188
   # Test workflows manually
   ```

2. **Staging endpoint:**
   - Create separate endpoint for testing
   - Use cheaper GPU (RTX 3060) for basic tests
   - Run representative workflows

3. **Production deployment:**
   - Once staging validated, deploy to production
   - Monitor logs closely for first few hours
   - Keep staging endpoint for future changes

4. **Manual test checklist:**
   - [ ] ComfyUI server starts successfully
   - [ ] Models detected from network volume
   - [ ] Simple workflow executes (txt2img)
   - [ ] Complex workflow executes (ControlNet, LoRA, etc.)
   - [ ] Image upload works (img2img)
   - [ ] S3 upload works (if configured)
   - [ ] Error handling (invalid workflow, missing model)
   - [ ] WebSocket reconnection (long workflow)

## FAQ

### Why is the network volume mandatory for ztex?

The ztex variant contains no pre-downloaded models. Without a network volume, there are no models available for ComfyUI to use, and all workflows will fail with "model not found" errors.

### Can I use ztex with models baked into a custom image?

Yes, but that defeats the purpose. If you want models in the image, use a different MODEL_TYPE (sdxl, sd3, etc.) or create a custom variant. The ztex philosophy is "network volume for everything."

### How do I update models without rebuilding?

Simply upload new models to your network volume:
1. Use RunPod web interface, S3 API, or temporary pod
2. Upload to `/runpod-volume/models/<subdirectory>/`
3. New workers will see the updated models immediately
4. Existing workers will pick up changes on next restart

### Will my first request be slower?

Possibly. ComfyUI loads models on first use, not at startup. If your model is on a network volume:
- First load: 5-30 seconds (depends on model size and network speed)
- Subsequent loads: Instant (cached in worker memory)

This is the same for all variants (baked or network volume).

### Can I mix baked and network volume models?

Yes! ComfyUI searches both locations:
1. `/comfyui/models/` (baked into image)
2. `/runpod-volume/models/` (network volume)

If a model exists in both, the baked version takes precedence.

### Why were tests removed?

This branch is focused on production deployment, not active development. The upstream repository maintains comprehensive tests. Removing tests reduces maintenance burden and simplifies the codebase for deployment-focused use cases.

If you need to run tests, reference the upstream repository or previous commits before the test removal.

### How do I contribute changes?

Since this is a fork:
1. Test changes thoroughly (see [Testing Your Changes](#testing-your-changes))
2. Document changes in commit messages
3. Update relevant documentation
4. Create pull request with detailed description

For upstream contributions, submit to the main worker-comfyui repository.

### What's the relationship to upstream?

This branch:
- **Based on**: worker-comfyui v5.3.0
- **Tracks**: Major upstream releases (rebase/merge)
- **Diverges**: Production-focused customizations (ztex, test removal)
- **Compatible**: All upstream features work (API, WebSocket, S3, etc.)

Think of it as "worker-comfyui v5.3.0, production edition."

### Can I use this with RunPod Pods (not serverless)?

Yes, but you'll need to adjust paths. Pods mount network volumes at `/workspace` instead of `/runpod-volume`. You would need to modify `src/extra_model_paths.yaml`:

```yaml
runpod_worker_comfy:
  base_path: /workspace  # Changed from /runpod-volume
  # ... rest unchanged
```

However, this worker is designed for **serverless**, not pods. For pods, consider using ComfyUI directly.

### How do I get support?

1. **Documentation**: Start with this guide and linked docs
2. **Troubleshooting**: See [Troubleshooting](#troubleshooting) section
3. **Debug mode**: Use `NETWORK_VOLUME_DEBUG=true` for diagnostics
4. **Logs**: Check worker logs in RunPod console
5. **Issues**: Report bugs with detailed logs and reproduction steps
6. **Upstream**: For core ComfyUI/RunPod issues, check upstream docs

---

**Related Documentation:**
- [Architecture Overview](architecture.md) - System design and components
- [Deployment Guide](deployment.md) - General deployment instructions
- [Network Volumes Guide](network-volumes.md) - Detailed volume configuration
- [Configuration Reference](configuration.md) - Environment variables
- [Customization Guide](customization.md) - Adding models and nodes
- [Technical Specification](spec.md) - Complete technical details
