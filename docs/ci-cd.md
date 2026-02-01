# CI/CD

This project includes GitHub Actions workflows to automatically build and deploy Docker images to Docker Hub.

> **📌 FluidStudio-Qwen Branch Note**
>
> This branch has simplified build process compared to upstream:
> - **Test execution removed** (commits: `3349260`, `aeb6bdf`)
> - **Build-only pipeline** (no test stage)
> - **ztex variant** builds in 5-10 minutes (no model downloads)
> - **Simplified Dockerfile** (commit: `559eb7a` removed wget commands)

## Automatic Deployment to Docker Hub with GitHub Actions

The repository contains two workflows located in the `.github/workflows` directory:

- [`dev.yml`](../.github/workflows/dev.yml): Creates the images (base, sdxl, sd3, flux, **ztex** variants) and pushes them to Docker Hub tagged as `<image_name>:dev` on every push to the `main` branch.
- [`release.yml`](../.github/workflows/release.yml): Creates the images and pushes them to Docker Hub tagged as `<image_name>:latest` and `<image_name>:<release_version>` (e.g., `worker-comfyui:5.3.0-ztex`). This workflow is triggered only when a new release is created on GitHub.

### FluidStudio-Qwen Build Variants

The CI/CD pipeline builds the following variants:

| Variant | Build Time | Image Size | Models Included | Notes |
|---------|------------|------------|-----------------|-------|
| `ztex` | 5-10 min | ~5-7 GB | None | **Default for this branch**, network volume mandatory |
| `base` | 5-10 min | ~5 GB | None | Clean ComfyUI, optional network volume |
| `sdxl` | 30-45 min | ~15 GB | SDXL + VAE | Upstream variant (still available) |
| `sd3` | 30-45 min | ~20 GB | SD3 Medium | Upstream variant (still available) |
| `flux1-dev` | 45-60 min | ~30 GB | FLUX.1 dev | Upstream variant (still available) |

**ztex variant** is built fastest due to no model downloads (commit: `559eb7a`).

### Configuration for Your Fork

If you have forked this repository and want to use these actions to publish images to your own Docker Hub account, you need to configure the following in your GitHub repository settings:

1.  **Secrets** (`Settings > Secrets and variables > Actions > New repository secret`):

    | Secret Name                | Description                                                                | Example Value       |
    | -------------------------- | -------------------------------------------------------------------------- | ------------------- |
    | `DOCKERHUB_USERNAME`       | Your Docker Hub username.                                                  | `your-dockerhub-id` |
    | `DOCKERHUB_TOKEN`          | Your Docker Hub access token with read/write permissions.                  | `dckr_pat_...`      |
    | `HUGGINGFACE_ACCESS_TOKEN` | Your READ access token from Hugging Face (required only for building SD3). | `hf_...`            |

2.  **Variables** (`Settings > Secrets and variables > Actions > New repository variable`):

    | Variable Name    | Description                                                                  | Example Value              |
    | ---------------- | ---------------------------------------------------------------------------- | -------------------------- |
    | `DOCKERHUB_REPO` | The target repository (namespace) on Docker Hub where images will be pushed. | `your-dockerhub-id`        |
    | `DOCKERHUB_IMG`  | The base name for the image to be pushed to Docker Hub.                      | `my-custom-worker-comfyui` |

With these secrets and variables configured, the actions will push the built images (e.g., `your-dockerhub-id/my-custom-worker-comfyui:dev`, `your-dockerhub-id/my-custom-worker-comfyui:1.0.0`, `your-dockerhub-id/my-custom-worker-comfyui:latest`) to your Docker Hub account when triggered.

## FluidStudio-Qwen Pipeline Changes

### Test Execution Removed

**Commits:** `3349260` ("remove tests"), `aeb6bdf` ("Remove test")

**What was removed:**
- Test execution stage in CI/CD pipeline
- `tests/` directory and all unit/integration tests
- `test_resources/` directory with test fixtures
- `.runpod/tests.json` configuration

**Impact on CI/CD:**
- ❌ **No automated testing** in pipeline
- ✅ **Faster builds** (no test execution time)
- ✅ **Simpler pipeline** (build → push only)

**Build-only pipeline:**
```
1. Checkout code
2. Set up Docker Buildx
3. Log in to Docker Hub
4. Build image (multi-platform if needed)
5. Push to Docker Hub
```

**Previous pipeline (upstream):**
```
1. Checkout code
2. Set up Docker Buildx
3. Build image
4. Run tests                 # ← REMOVED
5. Push to Docker Hub (if tests pass)
```

**Recommendation:**
- Manual testing required before deployment
- Use staging endpoint for validation
- See [Development Guide - Testing Your Changes](development.md#testing-your-changes)

### Build Optimization for ztex

**Commit:** `559eb7a` ("Fix: Do not download models")

**What changed:**
- Removed all `wget` model download commands from Dockerfile
- No `HUGGINGFACE_ACCESS_TOKEN` required for ztex builds
- Downloader stage simplified (lines 108-111 just echo)

**Benefits:**
- **5-10 minute builds** for ztex (vs 30-60 min for model variants)
- **No download failures** (network timeouts, HF auth issues)
- **Simpler secrets management** (no HF token for ztex)
- **Cleaner logs** (no lengthy download output)

**Build time comparison:**
| Variant | Upstream Time | FluidStudio-Qwen Time | Improvement |
|---------|---------------|----------------------|-------------|
| ztex | N/A (new) | 5-10 min | New variant |
| base | 5-10 min | 5-10 min | Same |
| sdxl | 30-45 min | 30-45 min | Same (if built) |
| flux1-dev | 45-60 min | 45-60 min | Same (if built) |

**Note:** Most deployments in FluidStudio-Qwen branch use ztex, so effective build time is now 5-10 minutes instead of 30-60 minutes.

## Local Build Testing

To test builds locally before pushing:

```bash
# Build ztex variant (fast)
docker build \
  --build-arg MODEL_TYPE=ztex \
  --platform linux/amd64 \
  -t worker-comfyui:test-ztex \
  .

# Verify image size (should be ~5-7 GB)
docker images worker-comfyui:test-ztex

# Test locally with docker-compose
docker-compose up

# Send test request
curl -X POST http://localhost:8000/runsync \
  -H "Content-Type: application/json" \
  -d @test_input.json
```

For more information, see the [FluidStudio-Qwen Guide](fluidstudio-qwen.md).
