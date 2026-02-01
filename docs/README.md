# Documentation Index

Welcome to the worker-comfyui-FluidStudio documentation. This directory contains comprehensive documentation for understanding, deploying, and customizing this ComfyUI serverless worker.

> **📌 FluidStudio-Qwen Branch**
>
> This is the **FluidStudio-Qwen** branch, a production-optimized fork of worker-comfyui v5.3.0.
>
> **If you're deploying with the ztex model type, START HERE:**
> - **[fluidstudio-qwen.md](fluidstudio-qwen.md)** - Comprehensive guide to this branch

## FluidStudio-Qwen Branch Documentation

### [fluidstudio-qwen.md](fluidstudio-qwen.md) - FluidStudio-Qwen Guide 🆕
**Comprehensive branch-specific guide** covering:
- Overview and purpose of this branch
- Key differences from upstream (ztex model type, test removal, etc.)
- Architecture and model loading flow
- Complete deployment guide for ztex
- Troubleshooting common issues
- Advanced configuration options
- Migration guide from other branches/variants
- FAQ

**When to read**: **START HERE** if you're deploying the FluidStudio-Qwen branch or using the ztex model type. This is the most important document for this branch.

## Overview Documents

### [spec.md](spec.md) - Technical Specification
**Complete technical reference** covering:
- Project architecture and design decisions
- Component breakdown and responsibilities
- Complete request flow with timing
- **No Database architecture** (stateless, file-based) 🆕
- Data structures and API specifications
- Integration points (RunPod, S3, ComfyUI)
- Error handling strategies
- Configuration reference
- Deployment architecture
- Performance characteristics
- Security considerations
- **FluidStudio-Qwen branch details** 🆕

**When to read**: You need a deep technical understanding of how the system works, data flows, or internal architecture.

**FluidStudio-Qwen updates**: Expanded section 1.4 with comprehensive branch details, added section 5 on "No Database" architecture, updated version history appendix.

### [architecture.md](architecture.md) - Architecture Overview
**High-level system design** including:
- System component diagram
- Request flow visualization
- Communication patterns
- Key design decisions
- **FluidStudio-Qwen branch specifics** (comprehensive section) 🆕
- Performance characteristics
- Quick references

**When to read**: You want a visual understanding of how components interact, or need a quick architectural reference.

**FluidStudio-Qwen updates**: Added extensive "FluidStudio-Qwen Branch Specifics" section covering ztex model type, test suite removal, directory changes, build simplification, and deployment considerations.

## Operational Guides

### [deployment.md](deployment.md) - Deployment Guide
**Step-by-step deployment instructions** for:
- **FluidStudio-Qwen (ztex) deployment** (comprehensive section) 🆕
- Creating RunPod templates
- Configuring endpoints
- Setting up environment variables
- **Network volume setup (mandatory for ztex)** 🆕
- Choosing GPU types
- Monitoring and scaling
- Verification and troubleshooting

**When to read**: You're ready to deploy the worker to RunPod platform.

**FluidStudio-Qwen updates**: Added complete "FluidStudio-Qwen (ztex) Deployment" section with step-by-step instructions for network volume setup, template creation, endpoint configuration, and verification.

### [configuration.md](configuration.md) - Configuration Reference
**Complete environment variable reference** including:
- General configuration
- Logging configuration
- WebSocket settings
- AWS S3 upload configuration
- Examples and use cases

**When to read**: You need to configure the worker for specific requirements (S3 upload, logging levels, etc.).

### [network-volumes.md](network-volumes.md) - Network Volumes Guide
**Network volume setup and usage** covering:
- **FluidStudio-Qwen: Network volume MANDATORY for ztex** 🆕
- Directory structure requirements
- **ztex-specific directory structure (loras/, unet/)** 🆕
- Model path configuration
- Troubleshooting volume issues
- Diagnostics tools (`NETWORK_VOLUME_DEBUG`)

**When to read**: You want to use network volumes for model storage instead of baking them into Docker images. **CRITICAL for ztex deployments.**

**FluidStudio-Qwen updates**: Emphasized mandatory network volume for ztex, added ztex-specific directory structure and debugging instructions.

## Customization Guides

### [customization.md](customization.md) - Customization Guide
**Adding custom models and nodes** via:
- Custom Dockerfile approach
- Network volume approach
- Installing ComfyUI custom nodes
- Downloading models

**When to read**: You need to add custom models, nodes, or modify the worker behavior.

### [development.md](development.md) - Development Guide
**Local development setup** including:
- Prerequisites and setup
- **Testing Your Changes (manual workflow)** 🆕
- Local API simulation with Docker Compose
- WSL2 setup for Windows

**When to read**: You want to develop, test, or debug the worker locally.

**FluidStudio-Qwen updates**: Test suite removed (commits: `3349260`, `aeb6bdf`). Added "Testing Your Changes" section with manual testing workflow and checklist.

## Process Documentation

### [ci-cd.md](ci-cd.md) - CI/CD Guide
**Automated build and release pipeline** covering:
- GitHub Actions workflows
- Docker build process
- Release versioning
- Build variants

**When to read**: You need to understand or modify the build/release process.

### [conventions.md](conventions.md) - Code Conventions
**Project conventions** including:
- Code style guidelines
- Commit message format
- Documentation standards

**When to read**: You're contributing code to the project.

### [planning/](planning/) - Planning Documents
**Feature planning and design documents** including:
- `001_websocket.md` - WebSocket implementation plan
- `002_restructure_docs.md` - Documentation restructuring
- `003_5090.md` - RTX 5090 support planning

**When to read**: You want to understand the history of major features or planned improvements.

## Additional Resources

### [acknowledgments.md](acknowledgments.md) - Acknowledgments
Credits and thanks to contributors and dependencies.

---

## Quick Navigation by Use Case

### "I want to deploy FluidStudio-Qwen (ztex) to RunPod" 🆕
1. **Start with [fluidstudio-qwen.md](fluidstudio-qwen.md)** - Complete guide
2. Follow [deployment.md - ztex section](deployment.md#fluidstudio-qwen-ztex-deployment)
3. **Set up network volume** (MANDATORY): [network-volumes.md](network-volumes.md)
4. Verify deployment with `NETWORK_VOLUME_DEBUG=true`
5. Troubleshoot: [fluidstudio-qwen.md - Troubleshooting](fluidstudio-qwen.md#troubleshooting)

### "I want to deploy upstream variants (sdxl, sd3, etc.) to RunPod"
1. Start with [README.md](../README.md) - Quickstart section
2. Review [deployment.md](deployment.md) for detailed steps
3. Configure using [configuration.md](configuration.md)
4. Optional: Set up [network-volumes.md](network-volumes.md)

### "I want to understand how it works"
1. Read [architecture.md](architecture.md) for high-level overview
2. Dive into [spec.md](spec.md) for detailed technical design
3. Review `handler.py` source code with newfound understanding

### "I want to add custom models/nodes"
1. Check [customization.md](customization.md) for approaches
2. For network volumes: [network-volumes.md](network-volumes.md)
3. For local testing: [development.md](development.md)

### "I want to develop locally"
1. Follow [development.md](development.md) setup instructions
2. Use Docker Compose for local testing
3. Review [configuration.md](configuration.md) for env vars

### "I want to contribute"
1. Read [conventions.md](conventions.md) for code style
2. Set up local environment: [development.md](development.md)
3. Review [ci-cd.md](ci-cd.md) for build process

### "I'm troubleshooting issues"
1. Check [configuration.md](configuration.md) for correct settings
2. Enable diagnostics: `NETWORK_VOLUME_DEBUG=true`
3. Review error handling in [spec.md](spec.md#8-error-handling)
4. Check logs with `COMFY_LOG_LEVEL=DEBUG`

---

## Documentation Hierarchy

```
docs/
├── README.md                   # This file - documentation index
├── fluidstudio-qwen.md         # FluidStudio-Qwen comprehensive guide 🆕
├── spec.md                     # Complete technical specification
├── architecture.md             # Architecture overview
│
├── deployment.md               # Deployment guide
├── configuration.md            # Configuration reference
├── network-volumes.md          # Network volumes guide
│
├── customization.md            # Customization guide
├── development.md              # Development guide
│
├── ci-cd.md                    # CI/CD documentation
├── conventions.md              # Code conventions
├── acknowledgments.md          # Credits
│
└── planning/                   # Feature planning docs
    ├── 001_websocket.md
    ├── 002_restructure_docs.md
    └── 003_5090.md
```

---

## Project Links

- **Upstream Repository**: https://github.com/runpod-workers/worker-comfyui
- **FluidStudio-Qwen Branch**: This repository (custom fork)
- **Docker Hub**: https://hub.docker.com/r/runpod/worker-comfyui
- **RunPod Platform**: https://www.runpod.io/
- **ComfyUI**: https://github.com/comfyanonymous/ComfyUI

---

**Last Updated**: 2026-01-31
**Branch**: FluidStudio-Qwen (based on worker-comfyui v5.3.0)
**Maintainer**: worker-comfyui-FluidStudio project
