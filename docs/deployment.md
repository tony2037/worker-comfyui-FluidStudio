# Deployment

This guide explains how to deploy the `worker-comfyui` as a serverless endpoint on RunPod, covering both pre-built official images and custom-built images.

> **FluidStudio-Qwen Branch Note**
>
> This branch uses the `ztex` model type by default, which **requires a network volume** (not optional). See the [FluidStudio-Qwen Deployment Section](#fluidstudio-qwen-ztex-deployment) below for detailed instructions specific to this branch.

## Deploying Pre-Built Official Images

This is the simplest method if the official images meet your needs.

### Create your template (optional)

- Create a [new template](https://runpod.io/console/serverless/user/templates) by clicking on `New Template`
- In the dialog, configure:
  - Template Name: `worker-comfyui` (or your preferred name)
  - Template Type: serverless (change template type to "serverless")
  - Container Image: Use one of the official tags, e.g., `runpod/worker-comfyui:<version>-sd3`. (Refer to the main [README.md](../README.md#available-docker-images) for available image tags and the current version).
  - Container Registry Credentials: Leave as default (images are public).
  - Container Disk: Adjust based on the chosen image tag, see [GPU Recommendations](#gpu-recommendations).
  - (optional) Environment Variables: Configure S3 or other settings (see [Configuration Guide](configuration.md)).
    - Note: If you don't configure S3, images are returned as base64. For persistent storage across jobs without S3, consider using a [Network Volume](customization.md#method-2-network-volume-alternative-for-models). If models on your network volume are not being detected, see [Network Volumes & Model Paths](network-volumes.md) for troubleshooting steps.
- Click on `Save Template`

### Create your endpoint

- Navigate to [`Serverless > Endpoints`](https://www.runpod.io/console/serverless/user/endpoints) and click on `New Endpoint`
- In the dialog, configure:

  - Endpoint Name: `comfy` (or your preferred name)
  - Worker configuration: Select a GPU that can run the model included in your chosen image (see [GPU recommendations](#gpu-recommendations)).
  - Active Workers: `0` (Scale as needed based on expected load).
  - Max Workers: `3` (Set a limit based on your budget and scaling needs).
  - GPUs/Worker: `1`
  - Idle Timeout: `5` (Default is usually fine, adjust if needed).
  - Flash Boot: `enabled` (Recommended for faster worker startup).
  - Select Template: `worker-comfyui` (or the name you gave your template).
  - (optional) Advanced: If you are using a Network Volume, select it under `Select Network Volume`. See the [Customization Guide](customization.md#method-2-network-volume-alternative-for-models). For detailed model path layout and debugging tips, see [Network Volumes & Model Paths](network-volumes.md).

- Click `deploy`
- Your endpoint will be created. You can click on it to view the dashboard and find its ID.

### GPU recommendations (for Official Images)

| Model                     | Image Tag Suffix | Minimum VRAM Required | Recommended Container Size | Network Volume Required |
| ------------------------- | ---------------- | --------------------- | -------------------------- | ----------------------- |
| Stable Diffusion XL       | `sdxl`           | 8 GB                  | 15 GB                      | Optional                |
| Stable Diffusion 3 Medium | `sd3`            | 5 GB                  | 20 GB                      | Optional                |
| FLUX.1 Schnell            | `flux1-schnell`  | 24 GB                 | 30 GB                      | Optional                |
| FLUX.1 dev                | `flux1-dev`      | 24 GB                 | 30 GB                      | Optional                |
| Base (No models)          | `base`           | N/A                   | 5 GB                       | Optional                |
| **ztex (FluidStudio)**    | **`ztex`**       | **Model-dependent**   | **10 GB**                  | **⚠️ MANDATORY**        |

_Note: Container sizes are approximate and might vary slightly. Custom images will vary based on included models/nodes._

**FluidStudio-Qwen Branch**: The `ztex` variant is the default for this branch. It contains NO pre-downloaded models and requires a network volume with models pre-uploaded.

## Deploying Custom Setups

If you have created a custom environment using the methods in the [Customization Guide](customization.md), here's how to deploy it.

> [!TIP] > **Want to skip the manual setup?**
>
> [ComfyUI-to-API](https://comfy.getrunpod.io) automatically generates a GitHub repository with a custom Dockerfile from your ComfyUI workflow. You can then deploy it using [Method 2: GitHub Integration](#method-2-deploying-via-runpod-github-integration) below with no manual Docker building required. See the [ComfyUI-to-API Documentation](https://docs.runpod.io/community-solutions/comfyui-to-api/overview) for details.

### Method 1: Manual Build, Push, and Deploy

This method involves building your custom Docker image locally, pushing it to a registry, and then deploying that image on RunPod.

1.  **Write your Dockerfile:** Follow the instructions in the [Customization Guide](customization.md#method-1-custom-dockerfile-recommended) to create your `Dockerfile` specifying the base image, nodes, models, and any static files.
2.  **Build the Docker image:** Navigate to the directory containing your `Dockerfile` and run:
    ```bash
    # Replace <your-image-name>:<tag> with your desired name and tag
    docker build --platform linux/amd64 -t <your-image-name>:<tag> .
    ```
    - **Crucially**, always include `--platform linux/amd64` for RunPod compatibility.
3.  **Tag the image for your registry:** Replace `<your-registry-username>` and `<your-image-name>:<tag>` accordingly.
    ```bash
    # Example for Docker Hub:
    docker tag <your-image-name>:<tag> <your-registry-username>/<your-image-name>:<tag>
    ```
4.  **Log in to your container registry:**
    ```bash
    # Example for Docker Hub:
    docker login
    ```
5.  **Push the image:**
    ```bash
    # Example for Docker Hub:
    docker push <your-registry-username>/<your-image-name>:<tag>
    ```
6.  **Deploy on RunPod:**
    - Follow the steps in [Create your template](#create-your-template-optional) above, but for the `Container Image` field, enter the full name of the image you just pushed (e.g., `<your-registry-username>/<your-image-name>:<tag>`).
    - If your registry is private, you will need to provide [Container Registry Credentials](https://docs.runpod.io/serverless/templates#container-registry-credentials).
    - Adjust the `Container Disk` size based on your custom image contents.
    - Follow the steps in [Create your endpoint](#create-your-endpoint) using the template you just created.

### Method 2: Deploying via RunPod GitHub Integration

RunPod offers a seamless way to deploy directly from your GitHub repository containing the `Dockerfile`. RunPod handles the build and deployment.

1.  **Prepare your GitHub Repository:** Ensure your repository contains the custom `Dockerfile` (as described in the [Customization Guide](customization.md#method-1-custom-dockerfile-recommended)) at the root or a specified path.
2.  **Connect GitHub to RunPod:** Authorize RunPod to access your repository via your RunPod account settings or when creating a new endpoint.
3.  **Create a New Serverless Endpoint:** In RunPod, navigate to Serverless -> `+ New Endpoint` and select the **"Start from GitHub Repo"** option.
4.  **Configure:**
    - Select the GitHub repository and branch you want to deploy (e.g., `main`).
    - Specify the **Context Path** (usually `/` if the Dockerfile is at the root).
    - Specify the **Dockerfile Path** (usually `Dockerfile`).
    - Configure your desired compute resources (GPU type, workers, etc.).
    - Configure any necessary [Environment Variables](configuration.md).
5.  **Deploy:** RunPod will clone the repository, build the image from your specified branch and Dockerfile, push it to a temporary registry, and deploy the endpoint.

Every `git push` to the configured branch will automatically trigger a new build and update your RunPod endpoint. For more details, refer to the [RunPod GitHub Integration Documentation](https://docs.runpod.io/serverless/github-integration).

---

## FluidStudio-Qwen (ztex) Deployment

This section provides **step-by-step deployment instructions** specifically for the `FluidStudio-Qwen` branch using the `ztex` model type.

> **Critical Requirement**
>
> The `ztex` variant contains **NO pre-downloaded models**. A network volume with models is **MANDATORY** (not optional). Deploying without a network volume will result in all jobs failing with "model not found" errors.

### Prerequisites

Before deploying, ensure you have:
- ✅ RunPod account with serverless access
- ✅ Models to upload (`.safetensors`, `.ckpt`, `.pt`, `.pth`, or `.bin` files)
- ✅ Basic understanding of ComfyUI model organization
- ✅ (Optional) S3-compatible storage credentials for large image outputs

### Step 1: Create and Prepare Network Volume

#### 1.1 Create Network Volume

1. Navigate to [RunPod Network Volumes](https://www.runpod.io/console/serverless/user/storage)
2. Click `+ Network Volume`
3. Configure:
   - **Name**: `comfyui-models` (or your preference)
   - **Size**: Based on your models
     - Small (1-2 models): 50 GB
     - Medium (5-10 models): 100-200 GB
     - Large (many models/variants): 500+ GB
   - **Region**: **MUST match your endpoint region** (critical!)
4. Click `Create`
5. Wait for volume to be ready (status: Active)

#### 1.2 Upload Models to Network Volume

You have several options for uploading models:

**Option A: RunPod Web Interface (Simplest)**
1. Go to your network volume page
2. Use the built-in file manager
3. Create directory structure:
   ```
   models/
   ├── checkpoints/
   ├── loras/
   ├── vae/
   └── unet/
   ```
4. Upload model files to appropriate directories
5. Verify file sizes match expectations (ensure uploads completed)

**Option B: S3-Compatible API (Recommended for Large Uploads)**

First, get your credentials from the network volume page, then:

```bash
# Install AWS CLI if not already installed
# brew install awscli  # macOS
# apt-get install awscli  # Ubuntu

# Configure AWS CLI with RunPod credentials
aws configure --profile runpod
# AWS Access Key ID: <from RunPod network volume page>
# AWS Secret Access Key: <from RunPod network volume page>
# Default region: us-east-1
# Default output format: json

# Upload a single model
aws s3 cp my-model.safetensors \
  s3://<NETWORK_VOLUME_ID>/models/checkpoints/my-model.safetensors \
  --profile runpod \
  --endpoint-url https://storage.runpod.io

# Upload entire directory
aws s3 sync ./local-models/ \
  s3://<NETWORK_VOLUME_ID>/models/ \
  --profile runpod \
  --endpoint-url https://storage.runpod.io
```

**Option C: Temporary Pod (For Initial Setup)**

1. Create a GPU pod with your network volume attached (mounted at `/workspace`)
2. Upload models via SSH, Jupyter, or rsync
3. Organize into proper structure under `/workspace/models/`
4. Terminate pod (network volume persists)

#### 1.3 Verify Directory Structure

**Required structure for ztex:**
```
/runpod-volume/
└── models/
    ├── checkpoints/          # SD 1.5, SDXL checkpoints
    │   └── my-model.safetensors
    ├── loras/                # LoRA models (required directory)
    │   └── style-lora.safetensors
    ├── vae/                  # VAE models
    │   └── vae-ft-mse.safetensors
    ├── unet/                 # FLUX, SD3 models (required directory)
    │   └── flux1-dev.safetensors
    ├── clip/                 # CLIP text encoders (for FLUX/SD3)
    │   ├── clip_l.safetensors
    │   └── t5xxl_fp16.safetensors
    ├── clip_vision/          # CLIP vision models
    ├── controlnet/           # ControlNet models
    ├── embeddings/           # Textual inversions
    └── upscale_models/       # Upscalers (RealESRGAN, etc.)
```

**Important notes:**
- The `models/` directory under network volume root is mandatory
- Only create subdirectories you actually need (empty folders are fine)
- File extensions must be: `.safetensors`, `.ckpt`, `.pt`, `.pth`, or `.bin`
- Files with other extensions (`.txt`, `.zip`) will be ignored

### Step 2: Create Serverless Template

1. Navigate to [Serverless Templates](https://runpod.io/console/serverless/user/templates)
2. Click `New Template`
3. Configure:

   **Basic Settings:**
   - **Template Name**: `worker-comfyui-ztex`
   - **Template Type**: `serverless` (change from default)
   - **Container Image**: `runpod/worker-comfyui:5.3.0-ztex`
     - Or use `latest-ztex` for most recent build
     - Or your custom image if you built one
   - **Container Disk**: `10 GB` (ztex base is small, ~5-7 GB)
   - **Container Registry Credentials**: Leave empty (public image)

   **Environment Variables (Optional but Recommended):**
   ```bash
   # Logging
   COMFY_LOG_LEVEL=INFO

   # Network volume debugging (enable for initial setup)
   NETWORK_VOLUME_DEBUG=true

   # S3 upload (if using S3 for large images)
   BUCKET_ENDPOINT_URL=https://s3.amazonaws.com
   BUCKET_ACCESS_KEY_ID=<your-key>
   BUCKET_SECRET_ACCESS_KEY=<your-secret>

   # Refresh worker after each job (optional, for debugging)
   # REFRESH_WORKER=true
   ```

4. Click `Save Template`

### Step 3: Create Serverless Endpoint

1. Navigate to [Serverless Endpoints](https://www.runpod.io/console/serverless/user/endpoints)
2. Click `New Endpoint`
3. Configure:

   **Basic Settings:**
   - **Endpoint Name**: `comfyui-ztex` (or your preference)
   - **Select Template**: `worker-comfyui-ztex` (created in Step 2)

   **Worker Configuration:**
   - **GPU Type**: Choose based on your models (see [GPU Selection Guide](#step-31-gpu-selection-guide))
   - **Active Workers**: `0` (scale to zero when idle)
   - **Max Workers**: `3` (adjust based on expected load and budget)
   - **GPUs/Worker**: `1`
   - **Idle Timeout**: `5` seconds (scale down quickly to save cost)

   **Advanced Settings (CRITICAL):**
   - ⚠️ **Select Network Volume**: Choose your `comfyui-models` volume
     - **This is MANDATORY for ztex**
     - Without this, all jobs will fail
   - **Flash Boot**: `enabled` (recommended for faster cold starts)
   - **FlashBoot Disk Size**: `10 GB` (default is fine)

4. Click `Deploy`
5. Wait for endpoint status to show "Ready" (green)

#### Step 3.1: GPU Selection Guide

| Model Type | VRAM Needed | Recommended GPU | RunPod GPU Options |
|------------|-------------|-----------------|-------------------|
| SD 1.5 | 4 GB | RTX 3060 Ti | RTX 3060 Ti, RTX 3070 |
| SDXL | 8 GB | RTX A5000 | RTX A5000, RTX 3090, RTX 4090 |
| SD3 Medium | 5 GB | RTX 4070 | RTX 4070, RTX 4080 |
| FLUX.1 Schnell | 16 GB | RTX A6000 | RTX A6000, A40 |
| FLUX.1 Dev | 24 GB | RTX A6000 / H100 | RTX A6000, H100, A100 |

**Cost optimization tip**: Start with cheaper GPUs (RTX 3060 Ti) for testing, then scale up once validated.

### Step 4: Verify Deployment

#### 4.1 Enable Debug Mode (First Deployment Only)

If not already set in template, add temporarily:
1. Go to endpoint → Manage → Edit
2. Add environment variable: `NETWORK_VOLUME_DEBUG=true`
3. Save (workers will restart)

#### 4.2 Send Test Request

```bash
# Replace <endpoint-id> and <your-api-key>
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
        },
        "5": {
          "class_type": "EmptyLatentImage",
          "inputs": {
            "width": 512,
            "height": 512,
            "batch_size": 1
          }
        },
        "6": {
          "class_type": "CLIPTextEncode",
          "inputs": {
            "text": "a beautiful landscape",
            "clip": ["4", 1]
          }
        },
        "7": {
          "class_type": "CLIPTextEncode",
          "inputs": {
            "text": "ugly, blurry",
            "clip": ["4", 1]
          }
        },
        "8": {
          "class_type": "VAEDecode",
          "inputs": {
            "samples": ["3", 0],
            "vae": ["4", 2]
          }
        },
        "9": {
          "class_type": "SaveImage",
          "inputs": {
            "images": ["8", 0],
            "filename_prefix": "ComfyUI"
          }
        }
      }
    }
  }'
```

**Replace `my-model.safetensors`** with your actual model filename.

#### 4.3 Check Logs

1. Go to endpoint → Logs tab
2. Look for network volume diagnostics:

**Success output:**
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

[5] Summary
    ✓ Models found on network volume!
======================================================================
```

**Expected response (if successful):**
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

#### 4.4 Disable Debug Mode (Production)

Once verified:
1. Edit endpoint environment variables
2. Remove `NETWORK_VOLUME_DEBUG=true` or set to `false`
3. Save (keeps logs clean in production)

### Step 5: Production Considerations

#### 5.1 Scaling Configuration

Adjust based on your traffic:
- **Low traffic** (< 10 requests/hour): Active: 0, Max: 1-2
- **Medium traffic** (10-100 requests/hour): Active: 0-1, Max: 3-5
- **High traffic** (> 100 requests/hour): Active: 1-2, Max: 10+

#### 5.2 Cost Optimization

- Use `REFRESH_WORKER=false` (default) to reuse workers
- Set Idle Timeout to 5 seconds (scale down fast)
- Use cheaper GPUs for testing/development
- Consider baking frequently-used models (hybrid approach)

#### 5.3 Monitoring

Monitor these metrics:
- **Job completion rate**: Should be > 95%
- **Average execution time**: Depends on model (track baseline)
- **Cold start time**: Should be < 30 seconds with Flash Boot
- **Error rate**: Check for "model not found" or CUDA OOM errors

### Troubleshooting

**Problem: "Model not found" errors**

Solution:
1. Enable `NETWORK_VOLUME_DEBUG=true`
2. Check logs for diagnostics
3. Common issues:
   - Network volume not attached → Edit endpoint, attach volume
   - Wrong directory structure → Models must be in `/runpod-volume/models/<type>/`
   - Wrong file extension → Use `.safetensors`, `.ckpt`, `.pt`, `.pth`, or `.bin`
   - Models in wrong subdirectory → Checkpoints in `checkpoints/`, not `loras/`

**Problem: Slow first request**

This is normal for ztex (models load from network volume). Subsequent requests are fast. If consistently slow:
- Check network volume region matches endpoint region
- Consider hybrid approach (bake frequently-used models)

**Problem: CUDA out of memory**

Solutions:
- Use GPU with more VRAM (see [GPU Selection Guide](#step-31-gpu-selection-guide))
- Reduce batch size in workflow
- Use FP8/quantized models if available

For more troubleshooting, see the [FluidStudio-Qwen Guide](fluidstudio-qwen.md#troubleshooting).

---

## Next Steps

- **Customize workflows**: See [Customization Guide](customization.md)
- **Add custom nodes**: See [Development Guide](development.md)
- **Configure environment**: See [Configuration Reference](configuration.md)
- **Understand architecture**: See [FluidStudio-Qwen Guide](fluidstudio-qwen.md)
