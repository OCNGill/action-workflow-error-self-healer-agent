```markdown
# Atlas Deployment Artifacts

This folder contains deployment scaffolds for running **Atlas** in a reproducible, GPU‑accelerated environment using ROCm.

## Contents
- **Dockerfile.rocm** — ROCm‑enabled Dockerfile (already created).
- **docker-compose.yml** — Compose scaffold for container orchestration.
- **atlas-deploy.md** — Teaching artifact documenting design decisions and usage.

## Usage

### Build the image
```bash
docker build -f Dockerfile.rocm -t atlas:rocm .
```

### Run with Compose
```bash
docker-compose up --build
```

### Stop containers
```bash
docker-compose down
```

## Notes
- Atlas remains a **subfolder** inside the container (`/app/Atlas`).
- ROCm devices `/dev/kfd` and `/dev/dri` are passed through for GPU access.
- Port `8000` is exposed by default; adjust if your workflow requires a different port.
- Extend volumes in `docker-compose.yml` if you want persistent logs or configs.

---
```