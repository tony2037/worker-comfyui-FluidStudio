# Customization

This guide covers methods for adding your own models, custom nodes, and static input files into a custom `worker-comfyui`.

> **📌 FluidStudio-Qwen Branch Note**
>
> This branch uses the `ztex` model type by default, which contains **NO pre-downloaded models**. Network volumes are the recommended approach for this branch. See [Customizing the ztex Model Type](#customizing-the-ztex-model-type) for branch-specific guidance.

> [!TIP]
>
> **Looking for the easiest way to deploy custom workflows?**
>
> [ComfyUI-to-API](https://comfy.getrunpod.io) automatically generates a custom Dockerfile and GitHub repository from your ComfyUI workflow, eliminating the manual setup described below. See the [ComfyUI-to-API Documentation](https://docs.runpod.io/community-solutions/comfyui-to-api/overview) for details.
>
> Use the manual methods below only if you need fine-grained control or prefer to manage everything yourself.

---

There are two primary methods for **manual** customization:

1.  **Custom Dockerfile (recommended for manual setup):** Create your own `Dockerfile` starting `FROM` one of the official `worker-comfyui` base images. This allows you to bake specific custom nodes, models, and input files directly into your image using `comfy-cli` commands. **This method does not require forking the `worker-comfyui` repository.**
2.  **Network Volume:** Store models on a persistent network volume attached to your RunPod endpoint. This is useful if you frequently change models or have very large models you don't want to include in the image build process.

## Method 1: Custom Dockerfile

> [!NOTE]
>
> This method does NOT require forking the `worker-comfyui` repository.

This is the most flexible and recommended approach for creating reproducible, customized worker environments.

1.  **Create a `Dockerfile`:** In your own project directory, create a file named `Dockerfile`.
2.  **Start with a Base Image:** Begin your `Dockerfile` by referencing one of the official base images. Using the `-base` tag is recommended as it provides a clean ComfyUI install with necessary tools like `comfy-cli` but without pre-packaged models.
    ```Dockerfile
    # start from a clean base image (replace <version> with the desired [release](https://github.com/runpod-workers/worker-comfyui/releases))
    FROM runpod/worker-comfyui:<version>-base
    ```
3.  **Install Custom Nodes:** Use the `comfy-node-install` (we had introduce our own cli tool here, as there is a [problem with comfy-cli not showing errors during installation](https://github.com/Comfy-Org/comfy-cli/pull/275)) command to add custom nodes by their name or URL, see [Comfy Registry](https://registry.comfy.org) to find the correct name. You can list multiple nodes.
    ```Dockerfile
    # install custom nodes using comfy-cli
    RUN comfy-node-install comfyui-kjnodes comfyui-ic-light
    ```
4.  **Download Models:** Use the `comfy model download` command to fetch models and place them in the correct ComfyUI directories.

    ```Dockerfile
    # download models using comfy-cli
    RUN comfy model download --url https://huggingface.co/KamCastle/jugg/resolve/main/juggernaut_reborn.safetensors --relative-path models/checkpoints --filename juggernaut_reborn.safetensors
    ```

> [!NOTE]
>
> Ensure you use the correct `--relative-path` corresponding to ComfyUI's model directory structure (starting with `models/<folder>`):
>
> checkpoints, clip, clip_vision, configs, controlnet, diffusers, embeddings, gligen, hypernetworks, loras, style_models, unet, upscale_models, vae, vae_approx, animatediff_models, animatediff_motion_lora, ipadapter, photomaker, sams, insightface, facerestore_models, facedetection, mmdets, instantid

5.  **Add Static Input Files (Optional):** If your workflows consistently require specific input images, masks, videos, etc., you can copy them directly into the image.

- Create an `input/` directory in the same folder as your `Dockerfile`.
- Place your static files inside this `input/` directory.
- Add a `COPY` command to your `Dockerfile`:

  ```Dockerfile
  # Copy local static input files into the ComfyUI input directory
  COPY input/ /comfyui/input/
  ```

- These files can then be referenced in your workflow using a "Load Image" (or similar) node pointing to the filename (e.g.,`my_static_image.png`).

Once you have created your custom `Dockerfile`, refer to the [Deployment Guide](deployment.md#deploying-custom-setups) for instructions on how to build, push and deploy your custom image to RunPod.

### Complete Custom `Dockerfile` Example

```Dockerfile
# start from a clean base image (replace <version> with the desired release)
FROM runpod/worker-comfyui:5.1.0-base

# install custom nodes using comfy-cli
RUN comfy-node-install comfyui-kjnodes comfyui-ic-light comfyui_ipadapter_plus comfyui_essentials ComfyUI-Hangover-Nodes

# download models using comfy-cli
# the "--filename" is what you use in your ComfyUI workflow
RUN comfy model download --url https://huggingface.co/KamCastle/jugg/resolve/main/juggernaut_reborn.safetensors --relative-path models/checkpoints --filename juggernaut_reborn.safetensors
RUN comfy model download --url https://huggingface.co/h94/IP-Adapter/resolve/main/models/ip-adapter-plus_sd15.bin --relative-path models/ipadapter --filename ip-adapter-plus_sd15.bin
RUN comfy model download --url https://huggingface.co/shiertier/clip_vision/resolve/main/SD15/model.safetensors --relative-path models/clip_vision --filename models.safetensors
RUN comfy model download --url https://huggingface.co/lllyasviel/ic-light/resolve/main/iclight_sd15_fcon.safetensors --relative-path models/diffusion_models --filename iclight_sd15_fcon.safetensors

# Copy local static input files into the ComfyUI input directory (delete if not needed)
# Assumes you have an 'input' folder next to your Dockerfile
COPY input/ /comfyui/input/
```

## Method 2: Network Volume

Using a Network Volume is primarily useful if you want to manage **models** separately from your worker image, especially if they are large or change often.

1.  **Create a Network Volume**:
    - Follow the [RunPod Network Volumes guide](https://docs.runpod.io/pods/storage/create-network-volumes) to create a volume in the same region as your endpoint.
2.  **Populate the Volume with Models**:
    - Use one of the methods described in the RunPod guide (e.g., temporary Pod + `wget`, direct upload, or the S3-compatible API) to place your model files into the correct ComfyUI directory structure **within the volume**.
    - For **serverless endpoints**, the network volume is mounted at `/runpod-volume`, and ComfyUI expects models under `/runpod-volume/models/...`. See [Network Volumes & Model Paths](network-volumes.md) for the exact structure and debugging tips.
      ```bash
      # Example structure inside the Network Volume (serverless worker view):
      # /runpod-volume/models/checkpoints/your_model.safetensors
      # /runpod-volume/models/loras/your_lora.pt
      # /runpod-volume/models/vae/your_vae.safetensors
      ```
    - **Important:** Ensure models are placed in the correct subdirectories (e.g., checkpoints in `models/checkpoints`, LoRAs in `models/loras`). If models are not detected, enable `NETWORK_VOLUME_DEBUG` as described in [Network Volumes & Model Paths](network-volumes.md).
3.  **Configure Your Endpoint**:
    - Use the Network Volume in your endpoint configuration:
      - Either create a new endpoint or update an existing one (see [Deployment Guide](deployment.md)).
      - In the endpoint configuration, under `Advanced > Select Network Volume`, select your Network Volume.

> [!NOTE]
>
> - When a Network Volume is correctly attached, ComfyUI running inside the worker container will automatically detect and load models from the standard directories (`/runpod-volume/models/...`) within that volume (for serverless workers). For directory mapping details and troubleshooting, see [Network Volumes & Model Paths](network-volumes.md).
> - This method is **not suitable for installing custom nodes**; use the Custom Dockerfile method for that.

---

## Customizing the ztex Model Type

The `ztex` model type (default for FluidStudio-Qwen branch) is designed for **network volume-based model management**. Here are several approaches for customizing it:

### Approach 1: Network Volume Only (Recommended)

**Use case:** Maximum flexibility, fastest builds

**How it works:**
- Use the stock `ztex` image (no customization)
- Store ALL models on network volume
- Update models without rebuilding images

**Setup:**
1. Use image: `runpod/worker-comfyui:5.3.0-ztex`
2. Create network volume
3. Upload models to `/runpod-volume/models/<type>/`
4. Attach network volume to endpoint

**Benefits:**
- ✅ Fast builds (5-10 minutes)
- ✅ Update models anytime (no rebuild)
- ✅ Share models across endpoints
- ✅ Small Docker image (~5-7 GB)

**Trade-offs:**
- ⚠️ Network volume mandatory
- ⚠️ Slightly slower first request (model loading from network)

**See:** [FluidStudio-Qwen Deployment Guide](deployment.md#fluidstudio-qwen-ztex-deployment)

### Approach 2: Hybrid (Some Baked, Some Network Volume)

**Use case:** Fast startup for common models, flexibility for custom models

**How it works:**
- Customize ztex Dockerfile to include frequently-used base models
- Use network volume for custom/experimental models
- ComfyUI searches both locations

**Dockerfile example:**
```dockerfile
# Start from ztex base
FROM runpod/worker-comfyui:5.3.0-ztex

# Switch to ComfyUI directory
WORKDIR /comfyui

# Download frequently-used base models
RUN comfy model download \
  --url https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned-emaonly.safetensors \
  --relative-path models/checkpoints \
  --filename v1-5-pruned-emaonly.safetensors

# Download VAE
RUN comfy model download \
  --url https://huggingface.co/stabilityai/sd-vae-ft-mse-original/resolve/main/vae-ft-mse-840000-ema-pruned.safetensors \
  --relative-path models/vae \
  --filename vae-ft-mse.safetensors

# Custom models will come from network volume
```

**Build and deploy:**
```bash
# Build custom image
docker build --platform linux/amd64 -t my-registry/worker-comfyui:custom-ztex .

# Push to registry
docker push my-registry/worker-comfyui:custom-ztex

# Deploy with network volume for custom models
# Baked models available immediately, network volume models also work
```

**Benefits:**
- ✅ Faster first startup (base models pre-loaded)
- ✅ Still flexible (network volume for custom models)
- ✅ Best of both worlds

**Trade-offs:**
- ⚠️ Larger image (add ~5-15 GB per model)
- ⚠️ Longer build times (add ~5-30 min per model)
- ⚠️ Need rebuild to change baked models

### Approach 3: Custom Model Type (Fully Baked)

**Use case:** Specific model set, simplest deployment, no network volume

**How it works:**
- Create new MODEL_TYPE variant with all models baked
- Deploy without network volume
- Similar to upstream sdxl/sd3/flux1-dev variants

**Dockerfile example:**
```dockerfile
# Build from source to customize MODEL_TYPE
FROM nvidia/cuda:12.6.3-cudnn-runtime-ubuntu24.04 AS base
# ... (copy base setup from main Dockerfile)

# Downloader stage with custom MODEL_TYPE
FROM base AS downloader
ARG HUGGINGFACE_ACCESS_TOKEN

# Create directories
RUN mkdir -p /comfyui/models/checkpoints /comfyui/models/loras /comfyui/models/vae

# Download all models for "my-custom" type
WORKDIR /comfyui
RUN comfy model download \
  --url https://huggingface.co/... \
  --relative-path models/checkpoints \
  --filename my-model.safetensors

RUN comfy model download \
  --url https://huggingface.co/... \
  --relative-path models/loras \
  --filename my-lora.safetensors

# Final stage
FROM base AS final
COPY --from=downloader /comfyui/models /comfyui/models
```

**Or modify existing Dockerfile:**
```dockerfile
# In Dockerfile, modify lines 108-111:
RUN if [ "$MODEL_TYPE" = "my-custom" ]; then \
      comfy model download --url https://... --relative-path models/checkpoints --filename ...; \
      comfy model download --url https://... --relative-path models/loras --filename ...; \
    fi
```

**Build:**
```bash
docker build \
  --build-arg MODEL_TYPE=my-custom \
  --build-arg HUGGINGFACE_ACCESS_TOKEN=<token> \
  --platform linux/amd64 \
  -t my-registry/worker-comfyui:my-custom \
  .
```

**Benefits:**
- ✅ Self-contained (no network volume needed)
- ✅ Faster startup (models pre-loaded)
- ✅ Reproducible (same models every deployment)

**Trade-offs:**
- ⚠️ Large image (30-50 GB with models)
- ⚠️ Long build times (30-60 minutes)
- ⚠️ Need rebuild to update models
- ⚠️ Less flexible

### Approach 4: Add Custom Nodes to ztex

**Use case:** Need specific ComfyUI custom nodes with network volume models

**Dockerfile example:**
```dockerfile
# Start from ztex base
FROM runpod/worker-comfyui:5.3.0-ztex

# Install custom nodes
RUN comfy-node-install \
  https://github.com/ltdrdata/ComfyUI-Manager.git \
  https://github.com/WASasquatch/was-node-suite-comfyui.git \
  https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite.git

# Models still come from network volume
```

**Benefits:**
- ✅ Custom nodes included
- ✅ Models still flexible (network volume)
- ✅ Reasonably fast builds

**Trade-offs:**
- ⚠️ Slightly larger image (add ~100-500 MB per node package)
- ⚠️ Still requires network volume for models

### Decision Matrix

| Approach | Build Time | Image Size | Network Volume | Flexibility | Best For |
|----------|------------|------------|----------------|-------------|----------|
| **Network Volume Only** | 5-10 min | ~5-7 GB | Mandatory | Highest | Production, frequent model changes |
| **Hybrid** | 15-30 min | ~15-25 GB | Optional | High | Common base + custom models |
| **Fully Baked** | 30-60 min | ~30-50 GB | Not needed | Low | Fixed model set, simple deployment |
| **Custom Nodes + Network Volume** | 10-15 min | ~7-10 GB | Mandatory | High | Need specific nodes + flexible models |

### Migration Example: From sdxl to ztex

**Before (sdxl variant):**
```bash
# Image: runpod/worker-comfyui:5.3.0-sdxl
# Models: Baked into image
# Network volume: Not used
```

**After (ztex with network volume):**
```bash
# 1. Extract SDXL models from image or download:
#    - sd_xl_base_1.0.safetensors
#    - sd_xl_vae.safetensors

# 2. Upload to network volume:
/runpod-volume/models/
├── checkpoints/
│   └── sd_xl_base_1.0.safetensors
└── vae/
    └── sdxl_vae.safetensors

# 3. Update endpoint:
#    - Image: runpod/worker-comfyui:5.3.0-ztex
#    - Attach network volume
#    - Test with NETWORK_VOLUME_DEBUG=true

# 4. Verify models detected:
#    Check logs for diagnostics
```

**Benefits of migration:**
- Can now add more models without rebuilding
- Faster deployments (smaller image)
- Share models across endpoints

For more details, see the [FluidStudio-Qwen Guide](fluidstudio-qwen.md).
