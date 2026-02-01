# Network Volumes & Model Paths

This document explains how to use RunPod **Network Volumes** with `worker-comfyui`, how model paths are resolved inside the container, and how to debug cases where models are not detected.

> **🔴 FluidStudio-Qwen Branch: Network Volume MANDATORY for ztex**
>
> The `ztex` model type (default for this branch) contains **NO pre-downloaded models**. A network volume with models is **REQUIRED** (not optional). Deploying ztex without a network volume will cause all jobs to fail with "model not found" errors.
>
> See the [FluidStudio-Qwen Guide](fluidstudio-qwen.md) for complete deployment instructions.

> **Scope**
>
> These instructions apply to **serverless endpoints** using this worker. Pods mount network volumes at `/workspace` by default, while serverless workers see them at `/runpod-volume`.

## Directory Mapping

For **serverless endpoints**:

- Network volume root is mounted at: `/runpod-volume`
- ComfyUI models are expected under: `/runpod-volume/models/...`

For **Pods**:

- Network volume root is mounted at: `/workspace`
- Equivalent ComfyUI model path: `/workspace/models/...`

If you use the S3-compatible API, the same paths map as:

- Serverless: `/runpod-volume/my-folder/file.txt`
- Pod: `/workspace/my-folder/file.txt`
- S3 API: `s3://<NETWORK_VOLUME_ID>/my-folder/file.txt`

## Expected Directory Structure

Models must be placed in the following structure on your network volume:

```text
/runpod-volume/
└── models/
    ├── checkpoints/      # Stable Diffusion checkpoints (.safetensors, .ckpt)
    ├── loras/            # LoRA files (.safetensors, .pt) [EXPLICIT in FluidStudio-Qwen]
    ├── vae/              # VAE models (.safetensors, .pt)
    ├── clip/             # CLIP models (.safetensors, .pt)
    ├── clip_vision/      # CLIP Vision models
    ├── controlnet/       # ControlNet models (.safetensors, .pt)
    ├── embeddings/       # Textual inversion embeddings (.safetensors, .pt)
    ├── upscale_models/   # Upscaling models (.safetensors, .pt)
    ├── unet/             # UNet models (FLUX, SD3) [EXPLICIT in FluidStudio-Qwen]
    └── configs/          # Model configs (.yaml, .json)
```

> **Note**
>
> Only create the subdirectories you actually need; empty or missing folders are fine.

> **FluidStudio-Qwen Branch Changes**
>
> This branch explicitly configures `loras/` and `unet/` directories in `src/extra_model_paths.yaml` (commit `1ff5df7`). These directories are now required for:
> - **loras/**: LoRA model support (explicitly configured)
> - **unet/**: FLUX.1 and SD3 model support (uses UNet instead of traditional checkpoints)
>
> See [fluidstudio-qwen.md](fluidstudio-qwen.md#directory-structure-changes) for details.

## Supported File Extensions

ComfyUI only recognizes files with specific extensions when scanning model directories.

| Model Type     | Supported Extensions                        |
| -------------- | ------------------------------------------- |
| Checkpoints    | `.safetensors`, `.ckpt`, `.pt`, `.pth`, `.bin` |
| LoRAs          | `.safetensors`, `.pt`                       |
| VAE            | `.safetensors`, `.pt`, `.bin`               |
| CLIP           | `.safetensors`, `.pt`, `.bin`               |
| ControlNet     | `.safetensors`, `.pt`, `.pth`, `.bin`       |
| Embeddings     | `.safetensors`, `.pt`, `.bin`               |
| Upscale Models | `.safetensors`, `.pt`, `.pth`               |

Files with other extensions (for example `.txt`, `.zip`) are **ignored** by ComfyUI’s model discovery.

## Common Issues

- **Wrong root directory**
  - Models placed directly under `/runpod-volume/checkpoints/...` instead of `/runpod-volume/models/checkpoints/...`.
- **Incorrect extensions**
  - Files named without one of the supported extensions are skipped.
- **Empty directories**
  - No actual model files present in `models/checkpoints` (or other folders).
- **Volume not attached**
  - Endpoint created without selecting a network volume under **Advanced → Select Network Volume**.

If any of the above is true, ComfyUI will silently fail to discover models from the network volume.

## Debugging with `NETWORK_VOLUME_DEBUG`

The worker exposes an opt‑in debug mode controlled via the `NETWORK_VOLUME_DEBUG` environment variable.

### When to Use

Enable this when:

- Models on your network volume are not appearing in ComfyUI
- You suspect the directory structure or file extensions are wrong
- You want to quickly verify what the worker can actually see on `/runpod-volume`

### How to Enable

1. Go to your serverless **Endpoint → Manage → Edit**.
2. Under **Environment Variables**, add:

   - `NETWORK_VOLUME_DEBUG=true`

3. Save and wait for workers to restart (or scale to zero and back up).
4. Send any request to your endpoint (even a minimal one) to trigger the diagnostics.

### Reading the Diagnostics

When enabled, each request prints a detailed report to the worker logs, for example:

```text
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

[5] Summary
    ✓ Models found on network volume!
======================================================================
```

If there is a problem, the diagnostics will instead highlight it, for example:

- Missing `models/` directory
- No valid model files in any subdirectory
- Files present but ignored due to wrong extensions

### Disabling Debug Mode

Once you have resolved your issue, disable diagnostics to keep logs clean:

- Remove the `NETWORK_VOLUME_DEBUG` environment variable, **or**
- Set `NETWORK_VOLUME_DEBUG=false`

This returns the worker to normal behavior without extra log noise.

## FluidStudio-Qwen (ztex) Specific Debugging

### Critical Checklist for ztex Deployments

If you're using the ztex model type (FluidStudio-Qwen branch), verify ALL of the following:

- [ ] **Network volume attached** to endpoint (Advanced → Select Network Volume)
- [ ] **Models uploaded** to network volume (not empty)
- [ ] **Correct directory structure**: `/runpod-volume/models/<type>/`
  - ❌ **WRONG**: `/runpod-volume/checkpoints/model.safetensors`
  - ✅ **CORRECT**: `/runpod-volume/models/checkpoints/model.safetensors`
- [ ] **Correct file extensions**: `.safetensors`, `.ckpt`, `.pt`, `.pth`, or `.bin`
  - ❌ **WRONG**: `model.zip`, `model.txt`
  - ✅ **CORRECT**: `model.safetensors`
- [ ] **Models in correct subdirectory**:
  - Checkpoints → `models/checkpoints/`
  - LoRAs → `models/loras/`
  - FLUX/SD3 → `models/unet/`
  - VAE → `models/vae/`
- [ ] **Region match**: Network volume and endpoint in same region
- [ ] **Debug mode enabled**: `NETWORK_VOLUME_DEBUG=true` for initial testing

### Common ztex Deployment Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| **Network volume not attached** | "Model not found" on all jobs | Edit endpoint → Advanced → Select Network Volume |
| **Wrong directory structure** | Models uploaded but not detected | Move models from `/runpod-volume/<type>/` to `/runpod-volume/models/<type>/` |
| **Empty volume** | Diagnostics show "No models found" | Upload models using RunPod web interface, S3 API, or temporary pod |
| **Wrong file extension** | Some models detected, others ignored | Rename `.zip` or `.txt` files to `.safetensors` or other supported extensions |
| **Models in wrong subdirectory** | Specific model type not found | Move checkpoints to `checkpoints/`, LoRAs to `loras/`, etc. |
| **Region mismatch** | Slow performance or mount issues | Recreate volume in same region as endpoint |

### ztex Diagnostics Example

**Successful ztex deployment** (with NETWORK_VOLUME_DEBUG=true):

```
======================================================================
NETWORK VOLUME DIAGNOSTICS (NETWORK_VOLUME_DEBUG=true)
======================================================================

[1] Checking extra_model_paths.yaml configuration...
    ✓ FOUND: /comfyui/extra_model_paths.yaml
    ✓ Configured paths: checkpoints, loras, vae, unet, clip, ...

[2] Checking network volume mount at /runpod-volume...
    ✓ MOUNTED: /runpod-volume

[3] Checking directory structure...
    ✓ FOUND: /runpod-volume/models
    ✓ FOUND: /runpod-volume/models/checkpoints
    ✓ FOUND: /runpod-volume/models/loras
    ✓ FOUND: /runpod-volume/models/unet

[4] Scanning model directories...

    checkpoints/:
      - sdxl_base_1.0.safetensors (6.5 GB)

    loras/:
      - style-lora.safetensors (144.2 MB)
      - character-lora.safetensors (78.5 MB)

    unet/:
      - flux1-dev.safetensors (23.8 GB)

    vae/:
      - sdxl_vae.safetensors (334.6 MB)

    clip/:
      - clip_l.safetensors (246.1 MB)
      - t5xxl_fp16.safetensors (4.9 GB)

[5] Summary
    ✓ Models found on network volume!
    ✓ Total model files: 7
    ✓ Total size: ~35 GB
======================================================================
```

**Failed ztex deployment** (network volume not attached):

```
======================================================================
NETWORK VOLUME DIAGNOSTICS (NETWORK_VOLUME_DEBUG=true)
======================================================================

[1] Checking extra_model_paths.yaml configuration...
    ✓ FOUND: /comfyui/extra_model_paths.yaml

[2] Checking network volume mount at /runpod-volume...
    ✗ NOT MOUNTED: /runpod-volume

    ⚠️  CRITICAL: Network volume not detected!

    For ztex deployments, a network volume is MANDATORY.

    To fix:
    1. Go to your endpoint → Manage → Edit
    2. Scroll to Advanced → Select Network Volume
    3. Choose your network volume
    4. Save and wait for workers to restart

======================================================================
```

### Quick Debugging Steps for ztex

If jobs fail with "model not found":

1. **Enable debug mode:**
   ```bash
   # Add to endpoint environment variables:
   NETWORK_VOLUME_DEBUG=true
   ```

2. **Send test request** (any workflow)

3. **Check logs** for diagnostics output

4. **Common fixes:**
   - ✅ Network volume not attached → Attach in endpoint settings
   - ✅ Wrong directory → Move models to `/runpod-volume/models/<type>/`
   - ✅ Empty volume → Upload models
   - ✅ Wrong extension → Rename to `.safetensors` or `.ckpt`

5. **Disable debug mode** once fixed:
   ```bash
   # Remove or set to false:
   NETWORK_VOLUME_DEBUG=false
   ```

For more troubleshooting, see the [FluidStudio-Qwen Guide](fluidstudio-qwen.md#troubleshooting).


