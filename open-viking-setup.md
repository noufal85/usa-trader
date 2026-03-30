# USA Trader

OpenClaw AI agent setup with OpenViking memory backend.

## Prerequisites

- **OS**: Linux (Ubuntu/Debian recommended), macOS, or Windows (WSL)
- **Node.js**: v22+ (for OpenClaw)
- **Python**: 3.10+ (for OpenViking)
- **An OpenAI API key** (or another supported provider — see [OpenViking docs](https://github.com/volcengine/OpenViking))

> **Note**: Go and GCC are NOT required — the OpenViking pip package includes pre-built AGFS binaries.

---

## Step 1: Install OpenClaw

```bash
# Install via npm (or pnpm)
npm install -g openclaw

# Add Telegram channel
openclaw channels add --channel telegram --token "$TELEGRAM_BOT_TOKEN"

# Set gateway mode (required before first start)
# Edit ~/.openclaw/openclaw.json and add "mode": "local" under the "gateway" key

# Allow user services to run without active login (needed for both OpenClaw and OpenViking)
loginctl enable-linger $USER

# Install and start the gateway as a systemd service
openclaw gateway install
systemctl --user start openclaw-gateway.service

# Set up exec approvals for Telegram command execution
openclaw approvals allowlist add "/usr/bin/*"
openclaw approvals allowlist add "/usr/local/bin/*"
openclaw approvals allowlist add "~/.local/bin/*"
openclaw approvals allowlist add "~/openviking-env/bin/*"
```

Verify it's running:

```bash
openclaw gateway status
openclaw channels status
```

> **Docs**: https://docs.openclaw.ai

---

## Step 2: Install OpenViking (Memory Backend)

### 2.1 Create a Python virtual environment

```bash
# Install venv support if needed (Ubuntu/Debian)
sudo apt install -y python3-venv python3-pip

# Create and activate the venv
python3 -m venv ~/openviking-env
source ~/openviking-env/bin/activate
```

### 2.2 Install OpenViking

```bash
pip install openviking --upgrade --force-reinstall
```

### 2.3 Configure OpenViking

Create the config directory and file:

```bash
mkdir -p ~/.openviking ~/openviking_workspace
```

Create `~/.openviking/ov.conf`:

```json
{
  "storage": {
    "workspace": "/home/YOUR_USER/openviking_workspace"
  },
  "log": {
    "level": "INFO",
    "output": "stdout"
  },
  "embedding": {
    "dense": {
      "api_base": "https://api.openai.com/v1",
      "api_key": "YOUR_OPENAI_API_KEY",
      "provider": "openai",
      "dimension": 3072,
      "model": "text-embedding-3-large"
    },
    "max_concurrent": 10
  },
  "vlm": {
    "api_base": "https://api.openai.com/v1",
    "api_key": "YOUR_OPENAI_API_KEY",
    "provider": "openai",
    "model": "gpt-4o",
    "max_concurrent": 100
  }
}
```

> **Note**: Replace `YOUR_USER` and `YOUR_OPENAI_API_KEY` with your actual values.
> OpenViking also supports Volcengine (Doubao), Anthropic, Gemini, DeepSeek, and Ollama.
> See [provider docs](https://github.com/volcengine/OpenViking#2-model-preparation).
> **Security**: Store API keys in a `.env` file (e.g., `~/.env.openclaw`) with `chmod 600`, not hardcoded in configs.

### 2.4 Test the server manually

```bash
export OPENVIKING_CONFIG_FILE=~/.openviking/ov.conf

python3 -c "
from openviking.server.app import create_app
import uvicorn
app = create_app()
uvicorn.run(app, host='127.0.0.1', port=1933)
"
```

In another terminal:

```bash
curl http://127.0.0.1:1933/health
# Expected: {"status":"ok","healthy":true,"version":"...","user_id":"default"}
```

> **Important**: Kill the test server (Ctrl+C) before proceeding to Step 3. The server holds a lock on the workspace directory — the systemd service will fail if the test process is still running.

---

## Step 3: Set Up OpenViking as a Systemd Service

Create `~/.config/systemd/user/openviking.service`:

```ini
[Unit]
Description=OpenViking Context Database
After=network.target

[Service]
Type=simple
Environment=OPENVIKING_CONFIG_FILE=/home/YOUR_USER/.openviking/ov.conf
ExecStart=/home/YOUR_USER/openviking-env/bin/start-openviking.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

Create the wrapper script `~/openviking-env/bin/start-openviking.sh`:

```bash
#!/bin/bash
export OPENVIKING_CONFIG_FILE=/home/YOUR_USER/.openviking/ov.conf
exec /home/YOUR_USER/openviking-env/bin/python3 -c "
from openviking.server.app import create_app
import uvicorn
app = create_app()
uvicorn.run(app, host='127.0.0.1', port=1933)
"
```

Make it executable and enable the service:

```bash
chmod +x ~/openviking-env/bin/start-openviking.sh

# Reload, enable, and start (linger already enabled in Step 1)
systemctl --user daemon-reload
systemctl --user enable openviking.service
systemctl --user start openviking.service

# Verify
systemctl --user status openviking.service
```

> **Note**: Using a wrapper script instead of inline `-c` avoids systemd shell escaping issues.

---

## Step 4: Connect OpenViking to OpenClaw

### 4.1 Set up the memory directory structure

```bash
source ~/openviking-env/bin/activate
export OPENVIKING_CONFIG_FILE=~/.openviking/ov.conf

# Create memory directories
openviking mkdir /agent/memories
```

### 4.2 Store a memory

```bash
# Write content to a temp file
cat > /tmp/test-memory.md << 'EOF'
# Test Memory
This is a test memory to verify the pipeline works.
EOF

# Store it
openviking add-resource /tmp/test-memory.md --to "/agent/memories/test.md" --wait
```

### 4.3 Search memories

```bash
openviking search "test memory" --uri /agent -n 5
```

### 4.4 Other useful commands

```bash
# List stored memories
openviking ls /agent/memories

# Read a specific memory
openviking read /agent/memories/test.md

# View the full tree
openviking tree / -L 3
```

### 4.5 Update your OpenClaw workspace TOOLS.md

Add the following to your agent's `TOOLS.md` so it knows how to use OpenViking:

```markdown
## OpenViking (Memory Backend)

- **Server**: http://127.0.0.1:1933
- **Config**: ~/.openviking/ov.conf
- **Venv**: ~/openviking-env

### Quick Reference

source ~/openviking-env/bin/activate
export OPENVIKING_CONFIG_FILE=~/.openviking/ov.conf

openviking add-resource /path/to/file.md --to "/agent/memories/name.md" --wait
openviking search "query" --uri /agent -n 5
openviking read /agent/memories/name.md
openviking ls /agent/memories
openviking tree /agent -L 3
```

---

## Memory Structure

OpenViking organizes context in scopes:

| Scope | Purpose |
|-------|---------|
| `/agent/memories/` | Agent's own memories, learnings, session logs |
| `/user/` | User context — preferences, entities, long-term memory |
| `/resources/` | Knowledge base, documents, references |
| `/session/` | Per-conversation context (managed automatically) |

---

## Service Management

Both services are managed via systemd (user-level):

```bash
# OpenClaw Gateway
systemctl --user status openclaw-gateway
systemctl --user restart openclaw-gateway
systemctl --user stop openclaw-gateway
journalctl --user -u openclaw-gateway -f

# OpenClaw status/probe (uses the CLI against the running service)
openclaw gateway status
openclaw channels status

# OpenViking
systemctl --user status openviking
systemctl --user restart openviking
systemctl --user stop openviking
journalctl --user -u openviking -f
```

> **Note**: After updating `TOOLS.md` or `MEMORY.md`, restart the gateway for changes to take effect:
> `systemctl --user restart openclaw-gateway`

---

## Links

- [OpenClaw Docs](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenViking GitHub](https://github.com/volcengine/OpenViking)
- [OpenViking Docs](https://www.openviking.ai/docs)
