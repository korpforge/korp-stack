# Korpforge Stack

Docker stack for Korpforge: self-hosted OpenClaw gateway.

## Prerequisites

- Docker & Docker Compose v2+
- Copy `.env.example` to `.env` and fill in your values

By default, `OPENCLAW_IMAGE=ghcr.io/openclaw/openclaw:latest` pulls the official image.
To build a custom image from the local `Dockerfile`, unset or remove `OPENCLAW_IMAGE` from `.env`.

## Installation

```bash
# Copy configuration
cp .env.example .env

# Generate a gateway token
echo "OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)" >> .env

# Run the onboarding wizard (configures AI provider)
docker compose run --rm --no-deps openclaw-gateway \
  node dist/index.js onboard --mode local --no-install-daemon

# Configure gateway for Docker mode
docker compose run --rm --no-deps openclaw-gateway \
  node dist/index.js config set --batch-json \
  '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]},{"path":"gateway.http.endpoints.chatCompletions.enabled","value":true}]'

# Start the gateway
docker compose up -d openclaw-gateway
```

## Next steps

1. Install the **Korpforge** VS Code extension (`korpforge.korpforge`)
2. Invoke `@korp` in the chat to initialize the agent skills:
   - `AGENTS.md`, `HEARTBEAT.md`, `IDENTIFY.md`, `SOUL.md`, `TOOLS.md`, `USER.md`
3. Add personal skills as needed via `@korp /skill add <name>`

## Usage

- **Control UI**: http://localhost:18789
- **Health check**: http://localhost:18789/healthz

### CLI commands

```bash
# Status
docker compose run --rm openclaw-cli status

# Send a message to the agent
docker compose run --rm openclaw-cli agent --message "Hello"

# Check channels
docker compose run --rm openclaw-cli channels status

# Add a channel (e.g. Telegram)
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"
```

### Management

```bash
# Stop
docker compose down

# Logs
docker compose logs -f openclaw-gateway

# Restart
docker compose restart openclaw-gateway

# Update
docker compose pull && docker compose up -d
```

## Architecture

```
┌───────────────────────────────────────────────────┐
│  Developer workstation                            │
│                                                   │
│  ┌─────────────────┐                              │
│  │  VS Code @korp  │                              │
│  └────┬───┬───┬────┘                              │
│       │   │   │                                   │
│       │   │   └──── Piper TTS (native binary)     │
│       │   │                                       │
│       │   └──────── whisper-server (native binary)│
│       │              :9500                        │
│       ▼                                           │
│  ┌──────────────────┐                             │
│  │ OpenClaw Gateway │ :18789  (Docker)            │
│  └────────┬─────────┘                             │
│           │                                       │
│           ▼                                       │
│       LLM (remote or local)                       │
└───────────────────────────────────────────────────┘
```

## Voice (STT + TTS)

Voice runs via **native binaries** on the workstation (not Docker) to leverage hardware acceleration (Metal/NEON on macOS).

### Whisper STT

```bash
# Install whisper.cpp (macOS)
brew install whisper-cpp

# Download the model
mkdir -p ~/.korpforge/models
curl -L -o ~/.korpforge/models/ggml-medium.bin \
  "https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-medium.bin"

# Start the server
whisper-server --model ~/.korpforge/models/ggml-medium.bin --port 9500 --language fr
```

### Piper TTS

```bash
# Install piper
pip install piper-tts

# Download a French voice
mkdir -p ~/.korpforge/models/piper
curl -L -o ~/.korpforge/models/piper/fr_FR-siwis-medium.onnx \
  "https://huggingface.co/rhasspy/piper-voices/resolve/main/fr/fr_FR/siwis/medium/fr_FR-siwis-medium.onnx"
curl -L -o ~/.korpforge/models/piper/fr_FR-siwis-medium.onnx.json \
  "https://huggingface.co/rhasspy/piper-voices/resolve/main/fr/fr_FR/siwis/medium/fr_FR-siwis-medium.onnx.json"
```

### sox (audio recording)

```bash
brew install sox   # macOS
apt install sox    # Linux
```
