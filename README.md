# VoxSales AI Voice Agent

A fully integrated, production-grade AI voice agent solution that connects real-time speech-to-speech AI with a live CRM automation pipeline. The system enables instant voice conversations with prospects, executes real-time function tools during calls, and synchronizes post-call analytics back to Airtable and Telegram — all from a single browser-accessible cockpit.

## Architecture

```
PSTN Call (Telnyx)
    │
    ▼
Voice Bridge (Port 8085)
    │
    ├── AssemblyAI Voice Agent (WebSocket)
    │       ├── STT: 16kHz PCM input
    │       ├── LLM: Mistral-large-latest (via AssemblyAI)
    │       └── TTS: 24kHz PCM output → anna (female, English)
    │
    ├── Browser Cockpit (WebSocket /media/browser)
    │       ├── PcmPlayer: 24kHz Web Audio scheduling
    │       └── downsampleTo16k: hardware rate → 16kHz
    │
    ├── Function Tools (during call)
    │       ├── book_demo → Telegram alert + cockpit UI update
    │       └── qualify_lead → BANT capture + cockpit UI update
    │
    └── Post-Call Sync
            └── n8n Webhook → Mistral BANT analysis → Airtable → Telegram
```

## Components

### Bridge (`bridge/`)
- **server.js** — Express + dual WebSocket server. Routes audio between Telnyx, AssemblyAI, and browser. Executes function tools. Syncs post-call to n8n.
- **index.html** — Browser cockpit with Web Audio PCM player, 16kHz downsampling, live dashboard with BANT intelligence.
- **package.json** — Dependencies: `express`, `ws`, `axios`, `dotenv`, `mime-types`, `@ricky0123/vad-web`

### n8n Workflow (`n8n/workflow.json`)
21-node automation workflow:
- Schedule trigger (every 5 minutes)
- Airtable lead search and routing
- Telegram notifications
- Mistral BANT analysis
- Post-call CRM sync

### Deployment (`deployment/`)
- **docker-compose.yml** — Traefik + n8n Docker stack with TLS via Let's Encrypt
- **telnyx-assembly-bridge.service** — systemd unit with OOM protection, CPU priority, EnvironmentFile

## Quick Start

### 1. Configure
```bash
cp bridge/.env.example bridge/.env
# Edit .env with your API keys and configuration
```

### 2. Install dependencies
```bash
cd bridge
npm install
```

### 3. Start the bridge
```bash
sudo systemctl start telnyx-assembly-bridge
# Or directly:
node server.js
```

### 4. Start n8n
```bash
cd deployment
docker compose up -d
```

### 5. Import the workflow
Import `n8n/workflow.json` into your n8n instance.

### 6. Access the cockpit
Open `https://your-domain.com/` and click "Answer Call".

## Configuration

| Variable | Required | Description |
|----------|----------|-------------|
| `ASSEMBLYAI_API_KEY` | Yes | AssemblyAI API key |
| `TELNYX_API_KEY` | Yes | Telnyx API key for voice/SIP |
| `TELNYX_CONNECTION_ID` | Yes | Telnyx connection ID |
| `TELNYX_PHONE_NUMBER` | Yes | Phone number in E.164 format |
| `PORT` | No | Bridge server port (default: 8085) |
| `PUBLIC_HOST` | No | Public hostname for WebSocket connections |
| `N8N_WEBHOOK_URL` | Yes | n8n webhook URL for post-call sync |

## Voice Configuration

The voice agent uses **anna**, AssemblyAI's highest-fidelity female voice:
- RMS: 3,428 (loudest among all 8 tested female voices)
- Peak headroom: 80.1% (no clipping)
- DC offset: 0.09 (negligible)
- Language: English only

## Function Tools

Two function tools are registered during the call:

| Tool | Purpose | Side Effects |
|------|---------|-------------|
| `book_demo` | Schedule product demonstration | Telegram alert + cockpit UI update |
| `qualify_lead` | Record BANT qualification data | Cockpit intelligence panel update |

Both tools use non-blocking execution to prevent audio pipeline interruption.

## Performance

| Metric | Value |
|--------|-------|
| Bridge memory footprint | 25 MB |
| Bridge CPU utilization | <1% |
| Audio latency (STT→LLM→TTS) | ~300ms P50 |
| Barge-in latency | <200ms |
| Tool execution latency | <50ms |
| Post-call sync latency | <3s |

## Security

- Credentials stored in `.env` with 600 permissions
- Never hardcoded in source code
- WebSocket auth via Bearer token
- n8n webhook uses local Docker network

## License

See LICENSE for details.
