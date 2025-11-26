# Docker Setup for Non-Headless Browser with X11 Forwarding

## Summary of Changes

This documents all modifications made to run browser-use in Docker with:
- GUI display via X11 forwarding to host
- Burp Suite proxy integration
- Persistent browser profile and data

---

## Files Modified

### 1. `Dockerfile`

**ARG defaults added** (line 40-43):
```dockerfile
ARG TARGETPLATFORM=linux/amd64
ARG TARGETOS=linux
ARG TARGETARCH=amd64
ARG TARGETVARIANT=
```

**Removed VERSION.txt read** (line 85):
```dockerfile
# Changed from:
RUN (echo "[i] Docker build for Browser Use $(cat /VERSION.txt) starting..." \
# To:
RUN (echo "[i] Docker build for Browser Use starting..." \
```

**Uncommented X11 dependencies** (lines 121-130):
```dockerfile
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked,id=apt-$TARGETARCH$TARGETVARIANT \
    echo "[+] Installing APT base system dependencies for $TARGETPLATFORM..." \
    && mkdir -p /etc/apt/keyrings \
    && apt-get update -qq \
    && apt-get install -qq -y --no-install-recommends \
        apt-transport-https ca-certificates apt-utils gnupg2 unzip curl wget grep \
        nano iputils-ping dnsutils jq \
        libnss3 libxss1 libasound2 libx11-xcb1 \
        fontconfig fonts-ipafont-gothic fonts-wqy-zenhei fonts-thai-tlwg fonts-khmeros fonts-symbola fonts-noto fonts-freefont-ttf \
        at-spi2-common fonts-liberation fonts-noto-color-emoji fonts-tlwg-loma-otf fonts-unifont libatk-bridge2.0-0 libatk1.0-0 libatspi2.0-0 libavahi-client3 \
        libavahi-common-data libavahi-common3 libcups2 libfontenc1 libice6 libnspr4 libsm6 libunwind8 \
        libxaw7 libxcomposite1 libxdamage1 libxfont2 \
        libxkbfile1 libxmu6 libxpm4 libxt6 x11-xkb-utils x11-utils xfonts-encodings \
        xfonts-scalable xfonts-utils xserver-common xvfb \
    && rm -rf /var/lib/apt/lists/*
```

**Note**: Removed `fonts-kacst` (not available in Debian 13)

---

### 2. `docker-compose.yml` (new file)

```yaml
services:
  browser-use:
    build: .
    image: browseruse:latest
    container_name: browser-use
    environment:
      - DISPLAY=${DISPLAY:-:0}
      - BROWSER_USE_HEADLESS=false
      # Burp proxy settings
      - BROWSER_USE_PROXY_URL=http://172.17.0.1:8080
      # Add your LLM API keys here:
      # - OPENAI_API_KEY=${OPENAI_API_KEY}
      # - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      # X11 socket for GUI forwarding
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      # Persistent data (browser profiles, downloads, screenshots, logs)
      - ./data:/data:rw
    # Use host network for simplest X11 access
    network_mode: host

    # Security: needed for Chrome sandbox in some setups
    security_opt:
      - seccomp:unconfined

    # Uncomment for GPU acceleration:
    # devices:
    #   - /dev/dri:/dev/dri

    stdin_open: true
    tty: true

    # Keep container running for MCP access via docker exec
    entrypoint: ["tail", "-f", "/dev/null"]
```

---

### 3. `browser-use-docker-mcp` (new file)

MCP wrapper script for Claude integration:
```bash
#!/bin/bash
exec docker exec -i browser-use browser-use --mcp
```
Make available: `cp browser-use-docker-mcp /usr/local/bin/browser-use-docker-mcp`
Make executable: `chmod +x /usr/local/bin/browser-use-docker-mcp`

---

### 4. `data/config.json` (persistent config)

```json
{
  "browser_profile": {
    "2c42f263-435e-4f56-98fd-11e851181e18": {
      "id": "2c42f263-435e-4f56-98fd-11e851181e18",
      "default": true,
      "created_at": "2025-11-26T13:42:10.694450",
      "headless": false,
      "user_data_dir": "/data/chrome-profile",
      "allowed_domains": null,
      "downloads_path": "/data/downloads",
      "window_size": {"width": 1024, "height": 768},
      "proxy": {
        "server": "http://172.17.0.1:8080",
        "bypass": "localhost,127.0.0.1",
        "username": null,
        "password": null
      }
    }
  },
  "llm": {
    "...": "..."
  },
  "agent": {
    "...": "..."
  }
}
```

---

### 5. `browser_use/mcp/server.py`

**Line 454** - Changed `use_vision` default from `True` to `False`:
```python
use_vision=arguments.get('use_vision', False),
```

**Line 482-484** - Disabled screenshot in `browser_get_state`:
```python
elif tool_name == 'browser_get_state':
    # Always disable screenshot (ignore client request)
    return await self._get_browser_state(include_screenshot=False)
```

---

### 6. `docker/README.md`

Added sections for:
- Running with GUI (Non-Headless Mode)
- Data Persistence

---

## Build & Run

```bash
# 1. Allow X11 from Docker
xhost +local:docker

# 2. Create data directory
mkdir -p data && chmod 777 data

# 3. Build (requires BuildKit)
DOCKER_BUILDKIT=1 docker compose build

# 4. Start container
docker compose up -d

# 5. Test browser with config
docker exec browser-use python -c "
import asyncio
from browser_use.browser.session import BrowserSession
from browser_use.browser.profile import BrowserProfile
from browser_use.config import load_browser_use_config, get_default_profile

async def test():
    config = load_browser_use_config()
    profile = BrowserProfile(**get_default_profile(config))
    s = BrowserSession(browser_profile=profile)
    await s.start()
    p = await s.get_current_page()
    await p.goto('http://burpsuite')
    await asyncio.sleep(60)

asyncio.run(test())
"
```

---

## Register MCP with Claude

```bash
claude mcp add --transport stdio browser-use -- /path/to/browser-use/browser-use-docker-mcp
```

---

## Data Persistence

| Host Path | Container Path | Contents |
|-----------|----------------|----------|
| `./data/chrome-profile/` | `/data/chrome-profile/` | Chrome profile (cookies, certs, extensions) |
| `./data/downloads/` | `/data/downloads/` | Downloaded files |
| `./data/config.json` | `/data/config.json` | Browser/LLM/Agent settings |

---

## Burp Suite Integration

1. Burp listens on host `172.17.0.1:8080`
2. Container accesses it via Docker bridge: `172.17.0.1:8080`
3. Install Burp CA cert in Chrome (persists in `./data/chrome-profile/`)
4. Navigate to `http://burpsuite` to download cert
5. Install cert: `chrome://certificate-manager/localcerts/usercerts`

---

## Troubleshooting

**Build fails with "unbound variable"**: Ensure ARG defaults are set in Dockerfile

**Build fails with "BuildKit required"**: Use `DOCKER_BUILDKIT=1 docker compose build`

**Package not found (exit code 100)**: Remove unavailable packages (e.g., `fonts-kacst`)

**Config not loading**: Direct Python usage requires manual config loading:
```python
from browser_use.config import load_browser_use_config, get_default_profile
config = load_browser_use_config()
profile = BrowserProfile(**get_default_profile(config))
```

**X11 connection refused**: Run `xhost +local:docker` on host
