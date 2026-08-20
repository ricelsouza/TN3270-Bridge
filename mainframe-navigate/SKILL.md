---
name: mainframe-navigate
description: Navigate a z/OS mainframe as an SME using the TN3270 Bridge MCP tools (mcp__tn3270-bridge__mainframe_*) when available, or the legacy REST API on port 7327 as a fallback. Handles login/logoff, ISPF navigation, dataset operations, USS shell, SDSF job monitoring, JCL submission, DB2 operator commands, RACF read-only inspection, IBM Health Checker (HZSPROC), IPCS dump analysis, and TLS-secured connections. Use when you need to interact with a mainframe terminal — read screens, send commands, check jobs, inspect security posture, or diagnose z/OS issues.
version: 2.2.0
author: ACI PS CMM
tags: [mainframe, tn3270, ispf, tso, uss, omvs, sdsf, z/os, tls, ssl, mcp, db2, racf, hzsproc, ipcs, rmf, smf, sysplex, sme]
category: installation
---

# /mainframe-navigate — TN3270 Mainframe Navigation

> Targets **TN3270 Bridge 2.0.0+** with native MCP tools (preferred). For 1.4.0–1.5.0 (REST + TLS) all patterns in this file still work. On 1.3.x or older, fall back to the legacy patterns at the bottom of this file.

**Safety first.** Every operation this skill drives is READ-ONLY unless the user has explicitly asked for a mutating change. See [`references/racf-inspect.md`](references/racf-inspect.md) and the safety guardrails file if present. Never issue `PE/PERMIT/ALU/ALTUSER/AU/ADDUSER/RDEF/RDEFINE/SETROPTS/SETR`, destructive `IDCAMS DELETE`, DB2 DDL/DML mutations, or console `C jobname / FORCE / V OFFLINE / Z EOD / SETPROG / SET SMF=` without explicit user confirmation.

## When to use

- Log into a z/OS mainframe and navigate ISPF panels
- Check datasets, members, JCL output
- Access Unix System Services (USS/OMVS)
- Monitor jobs via SDSF
- Execute TSO commands
- Perform automated health checks
- Inspect RACF profiles, DB2 operator state, Sysplex/CF policies (read-only)
- Diagnose ABENDs, dumps, and system-level events

## SME depth (progressive disclosure)

For depth beyond the Level-1 workflow below, read the matching file under `references/`:

| Need | File |
|---|---|
| Full MCP tool map (all 15 `mainframe_*` tools + when to use each) | [`references/mcp-tools.md`](references/mcp-tools.md) |
| SDSF depth (drill DDNAMEs, XDC export, filter by prefix, RMF Mon III via SM) | [`references/sdsf-depth.md`](references/sdsf-depth.md) |
| Submit JCL end-to-end (build → submit → poll → fetch spool → parse RC) | [`references/jcl-submit-loop.md`](references/jcl-submit-loop.md) |
| ISPF Edit / View / member management (primary + line commands, profile) | [`references/ispf-editor.md`](references/ispf-editor.md) |
| Dataset ops (LISTC/LISTDS/IDCAMS/IEBCOPY/IEBGENER/GDG, ISPF 3.2/3.3/3.4/3.14) | [`references/dataset-ops.md`](references/dataset-ops.md) |
| USS shell depth (ISHELL, chtag, iconv, BPXBATCH, tsocmd/mvscmd, env vars) | [`references/uss-depth.md`](references/uss-depth.md) |
| RACF read-only inspection (LU/LG/LISTDSD/RLIST/SEARCH + non-mutation rule) | [`references/racf-inspect.md`](references/racf-inspect.md) |
| DB2 z/OS operator commands (-DIS DATABASE/THREAD/BUFFERPOOL/UTIL/GROUP/LOG) | [`references/db2-operator.md`](references/db2-operator.md) |
| Console command matrix (D/F/S/P/C/K/V) with mutation blocklist | [`references/console-commands.md`](references/console-commands.md) |
| IBM Health Checker for z/OS (HZSPROC, HZSPRINT, exception triage) | [`references/health-checker.md`](references/health-checker.md) |
| ABEND triage playbook (S0C4/S806/S013/S222/SB37/S913) via TN3270 | [`references/abend-triage.md`](references/abend-triage.md) |
| IPCS dump analysis basics | [`references/ipcs-basics.md`](references/ipcs-basics.md) |

For deep z/OS System Programmer questions beyond navigation (Sysplex/CF/CFRM/SFM/ARM policy design, DFSMS ACS routines, WLM goal-mode tuning, IPCS control-block walking, IPL/parmlib, HZSPROC ecosystem), invoke the **`zos-sme`** agent when it is available in this session, or ask the user to load it.

## Preferred entry point — MCP tools (v2.0+)

When `mcp__tn3270-bridge__mainframe_status` is available in this session, prefer the MCP tools over the raw REST/curl patterns further down. They are typed, serialized per session, and safer for automation. Same underlying bridge, cleaner surface.

Minimum viable session:

```
mcp__tn3270-bridge__mainframe_status               # confirm bridge + connection + panelId
mcp__tn3270-bridge__mainframe_login  profile=SPS1 userid=C30T158
mcp__tn3270-bridge__mainframe_read_screen  nonblank=true
mcp__tn3270-bridge__mainframe_send  fieldName=OPTION text=3.4 aid=ENTER
mcp__tn3270-bridge__mainframe_send  labelLeft="Dsname Level" text=C30T158 aid=ENTER
mcp__tn3270-bridge__mainframe_wait  type=panel panelId=ISFPCU timeout=10000
mcp__tn3270-bridge__mainframe_read_all  scrollKey=PF8 stopText="BOTTOM OF DATA"
mcp__tn3270-bridge__mainframe_read_dataset  dsn=SYS1.PARMLIB member=IEASYS00
mcp__tn3270-bridge__mainframe_submit_job  dsn=USER.CMM.JCL member=COLLSMF
```

Full tool catalog (15 tools with signatures + examples): [`references/mcp-tools.md`](references/mcp-tools.md).

When the MCP tools are NOT present (headless CLI, bridge < 2.0, or a session that never loaded them), fall back to the REST patterns below — everything still works, just noisier.

---

## Prerequisites

TN3270 Bridge running in VS Code with an active connection.

```bash
curl -s http://localhost:7327/status
```

Expected: `"version":"1.5.0"` (or newer), `"connected":true`. If `"connection.tls"` is present, you are on a secure channel.

If older, see legacy section.

---

## TLS / SSL Connections (1.5.0+)

Most production z/OS hosts require TLS on **port 992**.

### Fast paths for connecting

```bash
# 1) Boolean shortcut — uses settings.json tls.* knobs
curl.exe -sS -X POST http://127.0.0.1:7327/connect \
  -H 'Content-Type: application/json' \
  -d '{"host":"mfssl.example.com","port":992,"tls":true}'

# 2) Per-call override (when settings aren't configured)
curl.exe -sS -X POST http://127.0.0.1:7327/connect \
  -H 'Content-Type: application/json' \
  -d '{"host":"...","port":992,"tls":{"enabled":true,"caBundle":"C:/certs/corp.pem"}}'

# 3) For end users: Command Palette → "TN3270 Bridge: Connect with TLS (Secure)"
#    → pick a preset (Modern strict / Corporate CA / Legacy / Dev self-signed)
```

### Confirming you got TLS

```bash
curl.exe -sS http://127.0.0.1:7327/status
```

Look at `connection.tls`:

```json
"tls": {
  "enabled": true,
  "version": "TLSv1.2",
  "cipher": "ECDHE-RSA-AES256-GCM-SHA384",
  "authorized": true,
  "peerCertificate": { "subject": "CN=...", "issuer": "...", "validTo": "..." }
}
```

If `authorized:false` and you didn't set `rejectUnauthorized:false`, the connect call failed — check `errorCode`.

### When connect fails — read errorCode

The bridge translates raw Node errors into actionable codes:

| `errorCode` | Likely fix |
|-------------|-----------|
| `TLS_SELF_SIGNED` | Dev cert. Use `tls.rejectUnauthorized:false` or add via `tls.caBundle`. |
| `TLS_CA_NOT_TRUSTED` | Corporate CA missing. Set `tls.caBundle` to the PEM path. |
| `TLS_CERT_EXPIRED` | Renew the host cert; or bypass with `rejectUnauthorized:false` for testing. |
| `TLS_HOSTNAME_MISMATCH` | Set `tls.sni` to the cert's CN, or connect by name not IP. |
| `TLS_VERSION_MISMATCH` | Wrong port (probably 23 not 992), or lower `tls.minVersion`. |
| `TLS_PROTOCOL_UNSUPPORTED` | Server too old. Try `tls.minVersion:"TLSv1.1"` or `"TLSv1"`. |
| `CONN_REFUSED` | Wrong host or port. |
| `CONN_RESET` | Plain socket sent to TLS port — or vice versa. |
| `CONN_TIMEOUT` | VPN? Firewall? |

### Profiles with TLS

```jsonc
"tn3270Bridge.loginProfiles": {
  "PROD-SECURE": {
    "host": "mfssl.prod.bank.com", "port": 992,
    "tls": { "enabled": true, "minVersion": "TLSv1.2", "caBundle": "C:\\certs\\corp.pem" },
    "steps": [ ... ]
  }
}
```

`POST /login {profile:"PROD-SECURE"}` brings up TLS automatically.

### Mutual TLS

```jsonc
"tn3270Bridge.tls.clientCert": "C:\\certs\\client.pem",
"tn3270Bridge.tls.clientKey":  "C:\\certs\\client.key",
"tn3270Bridge.tls.passphraseKey": "prod-key"
```

Store the passphrase: Command Palette → **TN3270 Bridge: Store TLS Passphrase** → identifier `prod-key` → enter passphrase. Stored in OS-encrypted SecretStorage, never in settings.json.

---

## API Reference (1.4.0)

Base URL: `http://localhost:7327`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/status` | Connection state, cursor, **panelId**, keyboardLocked |
| GET | `/screen` | Compact screen (default in 1.4.0) |
| GET | `/screen?nonblank=true` | Only non-blank rows — ~70% fewer tokens |
| GET | `/screen?compact=false` | Full 24×80 lines (legacy shape) |
| GET | `/screen/region?row=&col=&rows=&cols=` | Rectangular slice |
| GET | `/screen/fields` | Input fields |
| POST | `/send` | Send text/keys (JSON, form, or GET query) |
| GET | `/send?text=&aid=` | Same as POST, shell-safe |
| POST | `/key/<AID>` | Press just an AID key, no body |
| POST | `/cursor` | Move cursor to a position **without** pressing any key |
| POST | `/wait` | Wait for text/unlock/panel/screenChange |
| POST | `/exec` | Chain send/wait steps |
| POST | `/login` | Run a configured login profile |

### AID keys
`ENTER`, `PF1`–`PF24`, `PA1`–`PA3`, `CLEAR`, `SYSREQ`, `ATTN`

---

## Critical Rules (1.4.0)

1. **No sleeps.** `/send` and `/exec` now wait for the host to unlock the keyboard by default — the returned screen is the post-AID state. Pass `"waitForUnlock":false` only if you want fire-and-forget.
2. **Prefer semantic resolution over row/col.** Use `fieldName` (`OPTION`, `COMMAND`, `CURSOR`) or `labelLeft`/`labelAbove` in `/send`. For screens with no named fields (SDSF rows, ISPF editor, raw TSO), use `POST /cursor {row, col}` to reposition before pressing a key. The cursor position is included in every read buffer sent to the host — repositioning it locally is enough.
3. **Use `panelId` to confirm where you are.** Present in `/status` and `/send` response. `panelChanged:true` in the response is a clear signal a panel transition happened.
4. **Use `changedLines` instead of diffing two screens.** Saves tokens and is unambiguous.
5. **Hidden fields (passwords)** return empty content — by design. Use the `/login` profile flow so passwords never enter Claude's context.
6. **Concurrent calls are serialized** internally. Don't worry about racing on cursor.

---

## Calling From PowerShell (no JSON-quoting pain)

PowerShell mangles inline `-d '{...}'`. Pick the one that fits:

```powershell
# Option A — GET query string (read or single-key actions)
curl.exe -sS 'http://127.0.0.1:7327/send?text=3.4&aid=ENTER'
curl.exe -sS -X POST http://127.0.0.1:7327/key/PF3

# Option B — form-urlencoded (anything more complex than one field)
curl.exe -sS -X POST http://127.0.0.1:7327/send `
  -H 'Content-Type: application/x-www-form-urlencoded' `
  --data-urlencode 'labelLeft=Dsname Level' `
  --data-urlencode 'text=C30T158' `
  --data-urlencode 'aid=ENTER'

# Option C — Invoke-RestMethod with ConvertTo-Json (multi-step exec/login)
$body = @{ steps = @(
  @{ send = @{ fieldName='OPTION'; text='S'; aid='ENTER' } },
  @{ wait = @{ type='panel'; panelId='SDSF' } }
) } | ConvertTo-Json -Depth 5
Invoke-RestMethod http://127.0.0.1:7327/exec -Method POST `
  -ContentType 'application/json' -Body $body
```

From bash/zsh, plain `-d` works as documented.

---

## Login

### Preferred — configured profile

Configure once in VS Code settings (`tn3270Bridge.loginProfiles`):

```jsonc
"tn3270Bridge.loginProfiles": {
  "ACW2": {
    "host": "mfz900acw2", "port": 23,
    "steps": [
      { "waitFor": "Enter Logon Information",
        "fields": [
          {"labelLeft":"User","text":"${userid}"},
          {"labelLeft":"Password","text":"${password}"}
        ],
        "aid": "ENTER" },
      { "waitFor": "Application", "text": "1", "aid": "ENTER" },
      { "waitFor": "***", "aid": "ENTER" },
      { "waitFor": "Primary Option Menu" }
    ]
  }
}
```

Then:

```bash
curl.exe -sS -X POST http://127.0.0.1:7327/login \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'profile=ACW2' \
  --data-urlencode 'userid=C30T158' \
  --data-urlencode 'savePassword=true'
```

Password lookup order: explicit body → SecretStorage `tn3270.<profile>.password` → VS Code input box. After the first login with `savePassword=true`, subsequent calls need only `profile` and `userid`.

### Fallback — manual (no profile)

```bash
# At EMSP00 — type userid/password with labelLeft (no row/col)
curl.exe -sS -X POST http://127.0.0.1:7327/send \
  -H 'Content-Type: application/json' \
  -d '{"fields":[
        {"labelLeft":"User","text":"C30T158"},
        {"labelLeft":"Password","text":"***"}
      ],"aid":"ENTER"}'

# Pick TSO
curl.exe -sS 'http://127.0.0.1:7327/send?text=1&aid=ENTER'

# Dismiss banner
curl.exe -sS -X POST http://127.0.0.1:7327/key/ENTER
```

---

## ISPF Navigation

Every ISPF panel has an `Option ===>` field. The semantic name `OPTION` resolves to it automatically.

| Action | Call |
|--------|------|
| Choose option | `GET /send?fieldName=OPTION&text=3.4&aid=ENTER` |
| Go back | `POST /key/PF3` |
| Exit ISPF | `GET /send?fieldName=OPTION&text==X&aid=ENTER` |
| Scroll down/up | `POST /key/PF8` / `POST /key/PF7` |
| Confirm SDSF DA | `POST /exec { steps:[{send:{text:"S",aid:"ENTER"}},{send:{text:"DA",aid:"ENTER"}}] }` |

Verify position after a hop:

```bash
curl.exe -sS http://127.0.0.1:7327/status     # look at .screen.panelId
```

---

## Dataset List (ISPF 3.4)

No more padding, no more R10:C24 hardcode:

```bash
curl.exe -sS -X POST http://127.0.0.1:7327/send \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'labelLeft=Dsname Level' \
  --data-urlencode 'text=C30T158' \
  --data-urlencode 'aid=ENTER'
```

Scroll: `POST /key/PF8` (down), `POST /key/PF7` (up).

---

## SDSF (Job Monitoring)

```bash
# From ISPF — go to SDSF and check active users in one exec call
curl.exe -sS -X POST http://127.0.0.1:7327/exec -H 'Content-Type: application/json' -d '{
  "steps":[
    { "send": { "fieldName":"OPTION", "text":"S", "aid":"ENTER" } },
    { "wait": { "text":"SDSF MENU" } },
    { "send": { "fieldName":"COMMAND", "text":"DA", "aid":"ENTER" } }
  ]
}'
```

SDSF commands: `ST` (status), `H` (held), `O` (output), `DA` (active), `LOG` (syslog), `JC` (job classes), `ENQ` (enqueues).

---

## Unix System Services (OMVS)

```bash
# Get to USS shell
curl.exe -sS 'http://127.0.0.1:7327/send?fieldName=OPTION&text=6&aid=ENTER'
curl.exe -sS 'http://127.0.0.1:7327/send?text=OMVS&aid=ENTER'

# Issue commands — labelLeft handles cursor placement
curl.exe -sS -X POST http://127.0.0.1:7327/send \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'text=pwd' --data-urlencode 'aid=ENTER'

# Exit (two steps; OMVS asks confirmation)
curl.exe -sS -X POST http://127.0.0.1:7327/exec -H 'Content-Type: application/json' -d '{
  "steps":[
    { "send": { "text":"exit", "aid":"ENTER" } },
    { "send": { "aid":"ENTER" } }
  ]
}'
```

USS environment (ACW2/SPS1): home `/SPS1/home/<userid>`, shell `/bin/sh`, perl at `/rsusr/rocket/bin/perl`, ACI tools at `/syslpp/local/acitools` (curl, vim, egrep, diff, cvs).

---

## Logoff

```bash
curl.exe -sS -X POST http://127.0.0.1:7327/exec -H 'Content-Type: application/json' -d '{
  "steps":[
    { "send": { "fieldName":"OPTION", "text":"=X", "aid":"ENTER" } },
    { "send": { "text":"LOGOFF", "aid":"ENTER" } },
    { "send": { "text":"LOGOFF", "aid":"ENTER" } }
  ]
}'
```

---

## Patterns for Confidence

### Verify you're where you think you are

```bash
# After any navigation
curl.exe -sS http://127.0.0.1:7327/status | findstr panelId
```

If `panelId` is null, you're outside ISPF (EMSP00, EMSP01, banner, or a non-ISPF app).

### Wait without polling

```bash
# Wait until a specific panel shows up (e.g. after submit)
curl.exe -sS -X POST http://127.0.0.1:7327/wait -H 'Content-Type: application/json' \
  -d '{"type":"panel","panelId":"ISFPCU","timeout":10000}'

# Or just wait for the keyboard
curl.exe -sS -X POST http://127.0.0.1:7327/wait -H 'Content-Type: application/json' \
  -d '{"type":"keyboardUnlock","timeout":5000}'

# Or for any visible screen change
curl.exe -sS -X POST http://127.0.0.1:7327/wait -H 'Content-Type: application/json' \
  -d '{"type":"screenChange","timeout":5000}'
```

### Inspect only what changed

Every `/send` response includes:

```json
{
  "panelId": "ISR@PRI",
  "panelChanged": true,
  "changedLines": [{"row": 4, "content": "..."}]
}
```

Use `changedLines` directly — no need to re-fetch and diff yourself.

---

## Troubleshooting (1.4.0)

| Symptom | Fix |
|---------|-----|
| `"version":"1.3.0"` in `/status` | Bridge wasn't reloaded. In VS Code: Ctrl+Shift+P → "Developer: Reload Window" |
| `"Not connected"` | Use the **TN3270 Bridge** sidebar → Connect, or call `POST /connect` with `{host,port,luName?}` |
| `panelId` is null on a real ISPF panel | The host has PANELID OFF. Run `PANELID ON` in ISPF, or rely on text matches |
| `labelLeft` resolves to wrong field | Label is too generic. Use the longer surrounding text (e.g. `"Dsname Level"` not `"Level"`) or pass row/col explicitly |
| `/login` keeps asking for password | Pass `"savePassword":true` once; subsequent calls use SecretStorage |
| Old script breaks on 1.4.0 | Pass `"waitForUnlock":false` to restore 1.3.x fire-and-forget timing |

---

## Environment Details (ACW2/SPS1)

| Item | Value |
|------|-------|
| System | ACW2 / SPS1 — ACI CMM Development LPAR |
| z/OS | 2.5 |
| Session manager | EMSP (multi-application; option 1 = TSO) |
| Default user | C30T158 |
| TSO prefix | `$CMMUSER` |
| USS home | `/SPS1/home/<userid>` |
| Shell | `/bin/sh` |
| Perl | `/rsusr/rocket/bin/perl` (v5.24.0 for OS/390) |
| ACI tools | `/syslpp/local/acitools` |
| ISPF appl ID | `ISR` (panelId `ISR@PRI` = Primary Menu) |

---

## Legacy (Bridge ≤1.3.x)

If `/status` returns no `version` field or `"1.3.0"`, the bridge predates these features. Fall back to:

```bash
# Hard-coded positions, sleeps between calls
curl -s -X POST http://localhost:7327/send -H "Content-Type: application/json" \
  -d '{"text":"3.4","aid":"ENTER"}'
sleep 2
curl -s http://localhost:7327/screen
```

Upgrade path: install `tn3270-bridge-1.4.0.vsix` and reload window.

---

## Output Format

After each mainframe interaction, report:
1. **Panel/screen** you ended up on (use `panelId` when available)
2. **Key information** found (member contents, job status, command output, error code)
3. **Next action** if a multi-step task — or "done" with a one-line summary

Use markdown tables for tabular data (job lists, dataset lists, SDSF rows). Use code blocks (preserving column alignment) for raw screen excerpts.
