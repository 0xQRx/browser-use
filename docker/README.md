# Docker Setup for Browser-Use

This directory contains the optimized Docker build system for browser-use, achieving < 30 second builds.

## Quick Start

```bash
# Build base images (only needed once or when dependencies change)
./docker/build-base-images.sh

# Build browser-use
docker build -f Dockerfile.fast -t browseruse .

# Or use the standard Dockerfile (slower but self-contained)
docker build -t browseruse .
```

## Files

- `Dockerfile` - Standard self-contained build (~2 min)
- `Dockerfile.fast` - Fast build using pre-built base images (~30 sec)
- `docker/` - Base image definitions and build script
  - `base-images/system/` - Python + minimal system deps
  - `base-images/chromium/` - Adds Chromium browser
  - `base-images/python-deps/` - Adds Python dependencies
  - `build-base-images.sh` - Script to build all base images

## Performance

| Build Type | Time |
|------------|------|
| Standard Dockerfile | ~2 minutes |
| Fast build (with base images) | ~30 seconds |
| Rebuild after code change | ~16 seconds |

## Running with GUI (Non-Headless Mode)

To see the browser UI on your Linux host via X11 forwarding:

```bash
# 1. Allow X11 connections from Docker
xhost +local:docker

# 2. Build and run with docker-compose
docker-compose up --build

# 3. Or run directly with docker
docker run -it --rm \
  -e DISPLAY=$DISPLAY \
  -e BROWSER_USE_HEADLESS=false \
  -v /tmp/.X11-unix:/tmp/.X11-unix:rw \
  -v $(pwd)/data:/data:rw \
  --network host \
  browseruse:latest

# 4. Cleanup X11 permissions when done (optional)
xhost -local:docker
```

## Data Persistence

The `/data` volume persists all browser data between runs:

| Path | Contents |
|------|----------|
| `/data/profiles/` | Chrome user data (cookies, cache, localStorage, extensions) |
| `/data/downloads/` | Downloaded files |
| `/data/*.png`, `/data/*.pdf` | Screenshots and PDF exports |

Mount it with `-v $(pwd)/data:/data:rw` or use docker-compose.

**Note**: The container runs as user `browseruse` (UID 911). Ensure your host `./data` directory has appropriate permissions:
```bash
mkdir -p data && chmod 777 data
```
