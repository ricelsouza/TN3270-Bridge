# TN3270 Mainframe Bridge

**AI-operable TN3270 bridge for z/OS mainframe interaction.**

Version 2.0 is the corporate hardening release. The REST API is protected by a
SecretStorage-backed token, agents authenticate transparently through local
same-user IPC, and an MCP server is bundled for GitHub Copilot and Claude Code. See
`docs/AI_INTEGRATION.md`, `docs/SECURITY.md`, and `docs/MIGRATION_2.0.md`.

## Release 2.1.0

- Added a native VS Code MCP definition provider. GitHub Copilot discovers the
  mainframe tools directly from the installed extension; no `mcp.json` is needed.
- Added automatic user-scoped Claude Code registration during extension activation.
- Preserves an existing Claude MCP entry if it was not created by this extension.
- Updates extension-managed Claude registration automatically when the VSIX path changes.
- No token, clipboard, environment variable or setup command is required.
- VS Code still displays its mandatory MCP trust confirmation on first use.
- Requires VS Code 1.105 or newer; validated on VS Code 1.129 and Claude Code 2.1.168.

## Release 2.0.3

- Added JSON request bodies over `stdin` to `bridge-cli.js`.
- Eliminated PowerShell and RTK native-argument quoting failures for POST operations.
- Revalidated live read-only navigation on ISPF 3.4, SDSF `ST` and OMVS.
- Confirmed the session returns to `ISR@PRI` connected and unlocked.

## Release 2.0.2

- Removed token copy/paste, prompts and environment variables from normal use.
- Added a same-user IPC credential broker: named pipe on Windows and a `0600`
  Unix-domain socket on Linux/macOS.
- MCP now authenticates automatically with the Extension Host.
- Added `bridge-cli.js` for transparent authenticated terminal automation.
- Added E2E coverage proving REST, CLI and MCP work without a manually supplied token.
- Manual token copy remains available only for legacy direct REST scripts.

## Release 2.0.1

- Added an enforced release gate requiring matching README release notes.
- Added the permanent semantic-version and VSIX retention process.
- Preserved `tn3270-bridge-2.0.0.vsix`; this correction ships as a separate
  `tn3270-bridge-2.0.1.vsix` artifact.
- Revalidated TypeScript, eight automated tests, extension/MCP bundles and npm
  dependency audit.

## Release 2.0.0

- Added bearer-token REST authentication and blocked browser-originated calls.
- Added SecretStorage integration, hidden-field masking and diagnostic redaction.
- Added deterministic multi-session queues and transactional ISPF writes.
- Added CP037/CP500 support, corrected TN3270E model buffer reporting and MCP.
- Added E2E coverage for REST, TN3270 sockets and MCP stdio negotiation.

Connect to IBM mainframes, read 3270 screens, send keystrokes — all via a localhost HTTP API designed for AI agents (GitHub Copilot, Claude, ChatGPT) or terminal automation with `curl`.

![VS Code](https://img.shields.io/badge/VS%20Code-%5E1.93.0-blue)
![License](https://img.shields.io/badge/license-MIT-green)
![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey)

---

## Features

- **No External Binaries** — TypeScript TN3270 implementation with audited npm runtime dependencies
- **HTTP REST API** — Localhost server on port `7327` exposing all operations as JSON endpoints
- **MCP and REST** — Typed MCP tools for Copilot/Claude plus an authenticated localhost API
- **Full 3270 Protocol** — Telnet negotiation, TN3270E support, EBCDIC CP-037 translation, all 3270 orders (SBA, SF, SFE, RA, EUA, IC, PT, SA, MF, GE)
- **Screen Parsing** — Read screen text, identify input fields, track cursor position
- **Activity Bar Panel** — Real-time 3270 screen rendering in the VS Code sidebar
- **Editor Panel** — Full-size terminal view in the main editor area (`TN3270 Bridge: Open Terminal in Editor`)
- **Status Bar** — Connection status at a glance
- **Configurable** — Screen dimensions, code page, timeouts, auto-start

---

## Quick Start

### 1. Install

Install from `.vsix` file:
```
code --install-extension tn3270-bridge-2.1.0.vsix
```

### 2. Configure (optional)

Open VS Code Settings and search for `tn3270Bridge`:

| Setting | Default | Description |
|---------|---------|-------------|
| `tn3270Bridge.port` | `7327` | HTTP bridge port |
| `tn3270Bridge.autoStart` | `true` | Start bridge on VS Code launch |
| `tn3270Bridge.defaultHost` | `""` | Default mainframe hostname |
| `tn3270Bridge.defaultPort` | `23` | Default TN3270 port |
| `tn3270Bridge.codePage` | `cp037` | EBCDIC translation table |
| `tn3270Bridge.screenRows` | `24` | Screen rows (Model 2=24, Model 4=43) |
| `tn3270Bridge.screenCols` | `80` | Screen columns (Model 2=80, Model 5=132) |
| `tn3270Bridge.keyboardTimeout` | `30000` | Host response timeout (ms) |

### 3. Connect

MCP and the bundled CLI authenticate automatically. No token setup is required.

From the extension development folder:
```powershell
'{"host":"mainframe.example.com","port":992,"tls":true}' |
  npm run bridge -- POST /connect -
```

Or use the Command Palette: `TN3270 Bridge: Connect to Mainframe`

### 4. Interact

```bash
# Read the screen
curl http://localhost:7327/screen

# Send text and press Enter
curl -X POST http://localhost:7327/send \
  -H "Content-Type: application/json" \
  -d '{"text": "TSO MYUSER", "aid": "ENTER"}'

# Press PF3 (go back)
curl -X POST http://localhost:7327/send \
  -d '{"aid": "PF3"}'

# Wait for specific text to appear
curl -X POST http://localhost:7327/wait \
  -d '{"text": "READY", "timeout": 10000}'
```

---

## HTTP API Reference

The bridge exposes a REST API on `http://127.0.0.1:7327` (localhost only — no external access).

### `GET /status`

Returns bridge and connection status.

**Response:**
```json
{
  "bridge": "running",
  "connected": true,
  "connection": {
    "status": "connected",
    "host": "mainframe.example.com",
    "port": 23,
    "tn3270e": true,
    "deviceType": "IBM-3278-2-E"
  },
  "screen": {
    "rows": 24,
    "cols": 80,
    "cursorRow": 1,
    "cursorCol": 1,
    "fieldCount": 12,
    "keyboardLocked": false
  }
}
```

### `POST /connect`

Connect to a mainframe TN3270 service.

**Body:**
```json
{
  "host": "mainframe.example.com",
  "port": 23
}
```

**Response:**
```json
{
  "success": true,
  "message": "Connected to mainframe.example.com:23",
  "screen": ["line1...", "line2...", "...24 lines..."]
}
```

### `POST /disconnect`

Close the current connection.

**Response:**
```json
{ "success": true, "message": "Disconnected" }
```

### `GET /screen`

Read the current 3270 screen content.

**Response:**
```json
{
  "lines": ["line 1 content...", "...", "line 24 content..."],
  "text": "full screen as single string with newlines",
  "cursor": { "row": 5, "col": 20 },
  "keyboardLocked": false
}
```

### `GET /screen/fields`

List all screen fields (input and protected).

**Response:**
```json
{
  "fields": [
    { "index": 0, "row": 5, "col": 20, "length": 8, "content": "MYUSER", "protected": false },
    { "index": 1, "row": 6, "col": 20, "length": 8, "content": "", "protected": false }
  ],
  "inputFields": [...],
  "protectedFields": [...]
}
```

### `GET /screen/text?row=&col=&length=`

Read text at a specific screen position.

**Parameters:**
- `row` — Row number (1-based)
- `col` — Column number (1-based)
- `length` — Number of characters to read

**Response:**
```json
{ "text": "ISPF PRIMARY OPTION MENU", "row": 1, "col": 28, "length": 24 }
```

### `POST /send`

Send text and/or an AID key (Enter, PF1-PF24, PA1-PA3, Clear).

**Body:**
```json
{
  "text": "LOGON TSO",
  "aid": "ENTER",
  "row": 24,
  "col": 1,
  "field": null
}
```

| Field | Type | Description |
|-------|------|-------------|
| `text` | string | Text to type (optional) |
| `aid` | string | AID key: ENTER, PF1-PF24, PA1-PA3, CLEAR (default: ENTER) |
| `row` | number | Move cursor to row before typing (1-based, optional) |
| `col` | number | Move cursor to column before typing (1-based, optional) |
| `field` | number | Move cursor to field by index (optional, alternative to row/col) |

**Response:**
```json
{
  "success": true,
  "screen": ["...updated screen lines..."],
  "cursor": { "row": 1, "col": 1 }
}
```

### `POST /wait`

Wait until specific text appears on screen (polling internally).

**Body:**
```json
{
  "text": "READY",
  "timeout": 10000
}
```

**Response:**
```json
{
  "found": true,
  "screen": ["...current screen lines..."]
}
```

---

## AI Agent Usage Examples

### GitHub Copilot / Claude (via terminal)

```bash
# Connect to production mainframe
curl -s -X POST localhost:7327/connect -d '{"host":"mfprod01","port":23}' | jq .

# Login to TSO
curl -s -X POST localhost:7327/send -d '{"text":"TSO BATCHUSR","aid":"ENTER"}' | jq .screen
curl -s -X POST localhost:7327/wait -d '{"text":"Password"}' | jq .found
curl -s -X POST localhost:7327/send -d '{"text":"","aid":"ENTER","field":1}' | jq .

# Navigate to ISPF 3.4 (dataset list)
curl -s -X POST localhost:7327/send -d '{"text":"ISPF","aid":"ENTER"}' | jq .
curl -s -X POST localhost:7327/send -d '{"text":"3.4","aid":"ENTER"}' | jq .

# Read the dataset list screen
curl -s localhost:7327/screen | jq '.lines[]'

# Exit cleanly
curl -s -X POST localhost:7327/send -d '{"aid":"PF3"}' | jq .
curl -s -X POST localhost:7327/send -d '{"aid":"PF3"}' | jq .
curl -s -X POST localhost:7327/send -d '{"text":"LOGOFF","aid":"ENTER"}' | jq .
```

### Automation Script (Node.js)

```javascript
const BASE = 'http://localhost:7327';

async function mainframeLogin(host, user, password) {
  await fetch(`${BASE}/connect`, {
    method: 'POST',
    body: JSON.stringify({ host, port: 23 })
  });

  await fetch(`${BASE}/send`, {
    method: 'POST',
    body: JSON.stringify({ text: `TSO ${user}`, aid: 'ENTER' })
  });

  await fetch(`${BASE}/wait`, {
    method: 'POST',
    body: JSON.stringify({ text: 'Password', timeout: 10000 })
  });

  await fetch(`${BASE}/send`, {
    method: 'POST',
    body: JSON.stringify({ text: password, aid: 'ENTER', field: 1 })
  });

  const resp = await fetch(`${BASE}/wait`, {
    method: 'POST',
    body: JSON.stringify({ text: 'READY', timeout: 15000 })
  });

  return resp.json();
}
```

### PowerShell

```powershell
# Connect
Invoke-RestMethod -Uri "http://localhost:7327/connect" -Method POST `
  -Body '{"host":"mainframe.example.com","port":23}' -ContentType "application/json"

# Read screen
$screen = Invoke-RestMethod -Uri "http://localhost:7327/screen"
$screen.lines | ForEach-Object { Write-Host $_ }

# Send command
Invoke-RestMethod -Uri "http://localhost:7327/send" -Method POST `
  -Body '{"text":"TSO","aid":"ENTER"}' -ContentType "application/json"
```

---

## Commands

| Command | Description |
|---------|-------------|
| `TN3270 Bridge: Connect to Mainframe` | Open connection dialog |
| `TN3270 Bridge: Disconnect from Mainframe` | Close active connection |
| `TN3270 Bridge: Open Terminal in Editor` | Full-size terminal in the main editor area |
| `TN3270 Bridge: Read Current Screen` | Open screen content in a new editor tab |
| `TN3270 Bridge: Send Keys to Mainframe` | Interactive text + AID key input |
| `TN3270 Bridge: Toggle Bridge Server` | Start/stop the HTTP bridge |

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  AI Agent (Copilot, Claude, Scripts)                │
│  → curl http://localhost:7327/...                   │
└──────────────────────┬──────────────────────────────┘
                       │ HTTP (localhost only)
┌──────────────────────▼──────────────────────────────┐
│  VS Code Extension                                  │
│  ┌────────────────┐  ┌───────────────────────────┐  │
│  │ Bridge Server  │  │ Sidebar WebView            │  │
│  │ (port 7327)    │  │ (3270 screen rendering)   │  │
│  └───────┬────────┘  └───────────────────────────┘  │
│          │                                          │
│  ┌───────▼────────────────────────────────────────┐ │
│  │ TN3270 Client (Pure TypeScript)                │ │
│  │ • Telnet negotiation (BINARY, EOR, TN3270E)   │ │
│  │ • 3270 data stream parser (all orders)         │ │
│  │ • EBCDIC ↔ ASCII translation (CP-037)         │ │
│  │ • Screen buffer management (24×80)            │ │
│  │ • Field detection & modification tracking     │ │
│  └───────┬────────────────────────────────────────┘ │
└──────────┼──────────────────────────────────────────┘
           │ TCP/IP (TN3270 protocol)
┌──────────▼──────────────────────────────────────────┐
│  IBM z/OS Mainframe                                 │
│  VTAM → TSO/ISPF / CICS / IMS                      │
└─────────────────────────────────────────────────────┘
```

---

## Supported 3270 Features

### Protocol
- Telnet option negotiation (BINARY, EOR, TERMINAL_TYPE, TN3270E)
- TN3270E mode (with device type negotiation and functions)
- Basic TN3270 mode (fallback when TN3270E not available)
- CCW-format and SNA-format command recognition
- End-of-Record (EOR) framing

### Commands
- Write (W), Erase/Write (EW), Erase/Write Alternate (EWA)
- Read Buffer (RB), Read Modified (RM), Read Modified All (RMA)
- Erase All Unprotected (EAU)
- Write Structured Field (WSF) with Query Reply

### Orders
- Set Buffer Address (SBA)
- Start Field (SF), Start Field Extended (SFE)
- Insert Cursor (IC)
- Repeat to Address (RA), Erase Unprotected to Address (EUA)
- Program Tab (PT)
- Set Attribute (SA), Modify Field (MF)
- Graphic Escape (GE)

### AID Keys
- ENTER, CLEAR
- PF1 through PF24
- PA1, PA2, PA3

### Character Translation
- EBCDIC Code Page 037 (US/Canada/Western Europe)
- EBCDIC Code Page 500 (International)
- Full bidirectional translation table (256 entries)
- Null byte handling (display as spaces)

---

## Security

- **Localhost only** — The HTTP bridge binds exclusively to `127.0.0.1` and rejects non-local connections
- **Transparent authenticated API** — MCP/CLI obtain a 256-bit token from the Extension Host through same-user IPC
- **SecretStorage** — Login passwords and TLS key passphrases are encrypted by the operating system
- **No browser access** — CORS is disabled and requests carrying an `Origin` header are rejected
- **Masked output** — Hidden 3270 fields are redacted from full and positioned reads
- **TLS by default for production** — Use certificate validation and a corporate CA on port 992

See `docs/SECURITY.md` for the production policy and threat model.

---

## Troubleshooting

### Bridge not starting
Check if port 7327 is already in use. The extension tries up to 5 consecutive ports.
```bash
curl http://localhost:7327/status
```

### Connection timeout
- Verify network connectivity: `Test-NetConnection -ComputerName <host> -Port 23`
- Check firewall rules allow outbound TCP port 23
- Increase `tn3270Bridge.keyboardTimeout` in settings

### Screen appears empty
- Some mainframes require an initial ENTER or LOGON command
- Try: `curl -X POST localhost:7327/send -d '{"aid":"ENTER"}'`
- VTAM USS mode may show minimal text before application connection

### TN3270E not negotiated
- The extension falls back to basic TN3270 mode automatically
- Check `/status` endpoint for `tn3270e: false`
- Some mainframes only support basic TN3270

---

## Development

```bash
# Clone and install
git clone <repo-url> && cd tn3270-bridge
npm install

# Build
npm run bundle

# Watch mode (auto-rebuild on save)
npm run watch

# Package as .vsix
npx @vscode/vsce package --no-dependencies

# Install locally
code --install-extension tn3270-bridge-2.1.0.vsix
```

### Project Structure

```
tn3270-bridge/
├── src/
│   ├── extension.ts            # VS Code activation, commands, status bar
│   ├── bridge-server.ts        # HTTP REST API server
│   ├── tn3270-client.ts        # Core TN3270 protocol client
│   ├── tn3270-protocol.ts      # Constants, types, EBCDIC tables
│   ├── sidebar-provider.ts     # Sidebar WebView panel
│   └── editor-panel-provider.ts # Full-size editor panel
├── media/
│   ├── icon.png             # Extension marketplace icon
│   └── activitybar-icon.svg # Activity bar icon
├── out/
│   └── extension.js         # Bundled output (esbuild)
├── package.json             # Extension manifest
├── tsconfig.json            # TypeScript configuration
├── esbuild.js              # Build script
└── README.md               # This file
```

---

## Roadmap

- [x] Multiple concurrent sessions
- [x] CP037 and CP500
- [x] Dataset read/write through ISPF
- [x] MCP integration for Copilot and Claude Code
- [ ] Session recording/playback with secret redaction
- [ ] CP1047 and CP875 after validation against IBM reference vectors
- [ ] Light table (field map visualization)
- [ ] Keyboard shortcut passthrough in sidebar

---

## Author

**Ricel Leite**

---

## License

MIT © 2026 Ricel Leite
