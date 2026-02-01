# Development and Local Testing

This guide covers setting up your local environment for developing and testing the `worker-comfyui`.

> **📌 FluidStudio-Qwen Branch Note**
>
> The test suite has been **removed** from this branch (commits: `3349260`, `aeb6bdf`) to focus on production deployment. Manual testing is now required. See [Testing Your Changes](#testing-your-changes) below for the recommended workflow.

For manual testing, you can use the data from [`test_input.json`](../test_input.json) as a reference for workflow inputs.

## Setup

### Prerequisites

1.  Python >= 3.10
2.  `pip` (Python package installer)
3.  Virtual environment tool (like `venv`)

### Steps

1.  **Clone the repository** (if you haven't already):
    ```bash
    git clone https://github.com/runpod-workers/worker-comfyui.git
    cd worker-comfyui
    ```
2.  **Create a virtual environment**:
    ```bash
    python -m venv .venv
    ```
3.  **Activate the virtual environment**:
    - **Windows (Command Prompt/PowerShell)**:
      ```bash
      .\.venv\Scripts\activate
      ```
    - **macOS / Linux (Bash/Zsh)**:
      ```bash
      source ./.venv/bin/activate
      ```
4.  **Install dependencies**:
    ```bash
    pip install -r requirements.txt
    ```

### Setup for Windows (using WSL2)

Running Docker with GPU acceleration on Windows typically requires WSL2 (Windows Subsystem for Linux).

1.  **Install WSL2 and a Linux distribution** (like Ubuntu) following [Microsoft's official guide](https://learn.microsoft.com/en-us/windows/wsl/install). You generally don't need the GUI support for this.
2.  **Open your Linux distribution's terminal** (e.g., open Ubuntu from the Start menu or type `wsl` in Command Prompt/PowerShell).
3.  **Update packages** inside WSL:
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```
4.  **Install Docker Engine in WSL**:
    - Follow the [official Docker installation guide for your chosen Linux distribution](https://docs.docker.com/engine/install/#server) (e.g., Ubuntu).
    - **Important:** Add your user to the `docker` group to avoid using `sudo` for every Docker command: `sudo usermod -aG docker $USER`. You might need to close and reopen the terminal for this to take effect.
5.  **Install Docker Compose** (if not included with Docker Engine):
    ```bash
    sudo apt-get update
    sudo apt-get install docker-compose-plugin # Or use the standalone binary method if preferred
    ```
6.  **Install NVIDIA Container Toolkit in WSL**:
    - Follow the [NVIDIA Container Toolkit installation guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html), ensuring you select the correct steps for your Linux distribution running inside WSL.
    - Configure Docker to use the NVIDIA runtime as default if desired, or specify it when running containers.
7.  **Enable GPU Acceleration in WSL**:
    - Ensure you have the latest NVIDIA drivers installed on your Windows host machine.
    - Follow the [NVIDIA guide for CUDA on WSL](https://docs.nvidia.com/cuda/wsl-user-guide/index.html).

After completing these steps, you should be able to run Docker commands, including `docker-compose`, from within your WSL terminal with GPU access.

> [!NOTE]
>
> - It is generally recommended to run the Docker commands (`docker build`, `docker-compose up`) from within the WSL environment terminal for consistency with the Linux-based container environment.
> - Accessing `localhost` URLs (like the local API or ComfyUI) from your Windows browser while the service runs inside WSL usually works, but network configurations can sometimes cause issues.

## Testing Your Changes

> **⚠️ Test Suite Removed**
>
> The automated test suite (`tests/`, `test_resources/`, `.runpod/tests.json`) has been removed from the FluidStudio-Qwen branch (commits: `3349260`, `aeb6bdf`). This branch is focused on **production deployment**, not active development.
>
> **Why tests were removed:**
> - Core functionality tested in [upstream repository](https://github.com/runpod-workers/worker-comfyui)
> - Production deployments require end-to-end testing regardless
> - Reduces maintenance burden for deployment-focused fork
> - Simplifies codebase

### Recommended Manual Testing Workflow

Since automated tests are not available, follow this manual testing workflow:

#### 1. Local Testing with Docker Compose

Test changes locally before deploying to RunPod:

1. **Build and start services:**
   ```bash
   docker-compose up --build
   ```

2. **Access ComfyUI UI** (optional, for workflow creation):
   - Open http://localhost:8188 in browser
   - Create or modify workflows
   - Export workflow JSON

3. **Test handler directly:**
   ```bash
   # Send request to local worker
   curl -X POST http://localhost:8000/runsync \
     -H "Content-Type: application/json" \
     -d @test_input.json
   ```

4. **Verify output:**
   - Check response status
   - Verify images array
   - Check for errors array

#### 2. Testing Checklist

Before deploying changes to production, verify:

- [ ] **ComfyUI server starts successfully**
  - Check logs for startup errors
  - Verify port 8188 accessible

- [ ] **Models detected** (if using network volume):
  - Enable `NETWORK_VOLUME_DEBUG=true`
  - Check diagnostics output
  - Verify model paths correct

- [ ] **Simple workflow executes** (txt2img):
  - Create basic text-to-image workflow
  - Verify image generation
  - Check output format (base64/S3)

- [ ] **Complex workflow executes**:
  - Test with ControlNet (if applicable)
  - Test with LoRA (if applicable)
  - Test multi-step workflows

- [ ] **Image upload works** (img2img):
  - Test with base64 input images
  - Verify image processing
  - Check output correctness

- [ ] **S3 upload works** (if configured):
  - Set S3 environment variables
  - Verify S3 URL in response
  - Check presigned URL accessible

- [ ] **Error handling**:
  - Test invalid workflow (missing node)
  - Test missing model reference
  - Verify error messages clear

- [ ] **WebSocket reconnection** (long workflow):
  - Test workflow > 30 seconds
  - Check WebSocket stability
  - Verify reconnection works

#### 3. Staging Endpoint Testing

Before production deployment:

1. **Create staging endpoint:**
   - Separate from production
   - Use cheaper GPU (RTX 3060) for tests
   - Same configuration as production

2. **Run representative workflows:**
   - Test all workflow types you use
   - Verify outputs match expectations
   - Check performance (execution time)

3. **Monitor logs:**
   - Check for warnings/errors
   - Verify no unexpected behavior
   - Monitor resource usage

4. **Load testing** (optional):
   - Send multiple concurrent requests
   - Verify scaling works
   - Check for race conditions

#### 4. Production Deployment

Once staging validated:

1. **Deploy to production endpoint**
2. **Monitor logs closely** for first few hours
3. **Keep staging endpoint** for future changes
4. **Document any issues** encountered

### Testing Resources

**Sample workflows:**
- Reference upstream `test_resources/workflows/` (if available in git history)
- Export workflows from ComfyUI UI
- Use `test_input.json` as template

**Upstream tests** (for reference):
- View test suite in upstream repository: https://github.com/runpod-workers/worker-comfyui/tree/main/tests
- Useful for understanding expected behavior
- Can manually replicate test scenarios

**Debugging tools:**
- `NETWORK_VOLUME_DEBUG=true` - Model detection diagnostics
- `COMFY_LOG_LEVEL=DEBUG` - Verbose logging
- `REFRESH_WORKER=true` - Fresh worker per job (debugging)

## Local API Simulation (using Docker Compose)

For enhanced local development and end-to-end testing, you can start a local environment using Docker Compose that includes the worker and a ComfyUI instance.

> [!IMPORTANT]
>
> - This currently requires an **NVIDIA GPU** and correctly configured drivers + NVIDIA Container Toolkit (see Windows setup above if applicable).
> - Ensure Docker is running.

**Steps:**

1.  **Set Environment Variable (Optional but Recommended):**
    - While the `docker-compose.yml` sets `SERVE_API_LOCALLY=true` by default, you might manage environment variables externally (e.g., via a `.env` file).
    - Ensure the `SERVE_API_LOCALLY` environment variable is set to `true` for the `worker` service if you modify the compose file or use an `.env` file.
2.  **Start the services**:
    ```bash
    # From the project root directory
    docker-compose up --build
    ```
    - The `--build` flag ensures the image is built locally using the current state of the code and `Dockerfile`.
    - This will start two containers: `comfyui` and `worker`.

### Access the Local Worker API

- With the Docker Compose stack running, the worker's simulated RunPod API is accessible at: [http://localhost:8000](http://localhost:8000)
- You can send POST requests to `http://localhost:8000/run` or `http://localhost:8000/runsync` with the same JSON payload structure expected by the RunPod endpoint.
- Opening [http://localhost:8000/docs](http://localhost:8000/docs) in your browser will show the FastAPI auto-generated documentation (Swagger UI), allowing you to interact with the API directly.

### Access Local ComfyUI

- The underlying ComfyUI instance running in the `comfyui` container is accessible directly at: [http://localhost:8188](http://localhost:8188)
- This is useful for debugging workflows or observing the ComfyUI state while testing the worker.

### Stopping the Local Environment

- Press `Ctrl+C` in the terminal where `docker-compose up` is running.
- To ensure containers are removed, you can run: `docker-compose down`
