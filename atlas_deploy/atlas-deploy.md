# Atlas Deployment Notes (Teaching Artifact)

This document explains the rationale behind the deployment scaffolds for Atlas.

## Design Principles
- **Local first, container second**: Validate Atlas in LM Studio on HTPC before containerizing.
- **Subfolder integrity**: Atlas is always preserved as a subfolder, never renamed at repo root.
- **GPU acceleration**: ROCm runtime hooks (`/dev/kfd`, `/dev/dri`) ensure AMD GPU access.
- **Reproducibility**: Docker + Compose lock in a consistent runtime environment.

## Workflow
1. **Develop & Test**  
   - Run Atlas locally in LM Studio.  
   - Validate memory scaffolds, prompt bindings, and orchestration configs.  

2. **Containerize**  
   - Build the ROCm image with `docker build -f Dockerfile.rocm -t atlas:rocm .`.  
   - Use Compose for orchestration and persistent mounts.  

3. **Deploy**  
   - Run `docker-compose up --build`.  
   - Access Atlas on `localhost:8000` (or configured port).  

## Extending
- **Volumes**: Add mounts for configs, datasets, or checkpoints.  
- **Multi-agent orchestration**: Add more services to `docker-compose.yml`.  
- **CI/CD**: Integrate build/test steps into GitHub Actions for automated validation.  

## Teaching Artifact Purpose
This file is part of the **legacy documentation layer**. It explains not just *how* to deploy Atlas, but *why* the scaffolds are structured this way, ensuring future builders can extend and maintain the system with clarity.

---
