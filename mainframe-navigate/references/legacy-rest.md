# Legacy REST API Fallback (TN3270 Bridge < 2.0)

The TN3270 Bridge exposes a REST API on `http://localhost:7327`. When the MCP tools (`mcp__tn3270-bridge__mainframe_*`) are NOT loaded in this session, or the bridge is v1.4.0–1.5.0, or you are in a headless run, drive it via `curl`.

Same underlying bridge — just noisier and shell-quoting-prone.

---

## Confirm bridge is up

```bash
curl -s http://localhost:7327/status
```

Expected: `"version":"1.5.0"` (or `"2.2.0"`) + `"connected":true`. `panelId` in `screen` if in ISPF.

---

## TLS / SSL (Bridge 1.5.0+)

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

### Mutual TLS

```jsonc
"tn3270Bridge.tls.clientCert": "C:\\certs\\client.pem",
"tn3270Bridge.tls.clientKey":  "C:\\certs\\client.key",
"tn3270Bridge.tls.passphraseKey": "prod-key"
```

Store the passphrase: Command Palette → **TN3270 Bridge: Store TLS Passphrase** → identifier `prod-key`.

---

## REST endpoint catalog

Base URL: `http://localhost:7327`

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/status` | Connection state, cursor, panelId, keyboardLocked |
| GET | `/screen` | Compact screen (default in 1.4.0+) |
| GET | `/screen?nonblank=true` | Only non-blank rows |
| GET | `/screen?compact=false` | Full 24×80 lines |
| GET | `/screen/region?row=&col=&rows=&cols=` | Rectangular slice |
| GET | `/screen/fields` | Input fields |
| POST | `/send` | Send text/keys (JSON, form, or GET query) |
| GET | `/send?text=&aid=` | Same as POST, shell-safe |
| POST | `/key/<AID>` | Press just an AID key, no body |
| POST | `/cursor` | Move cursor without pressing any key |
| POST | `/wait` | Wait for text/unlock/panel/screenChange |
| POST | `/exec` | Chain send/wait steps |
| POST | `/login` | Run a configured login profile |
| POST | `/connect` | Explicit connect with host/port/tls |

### AID keys
`ENTER`, `PF1`–`PF24`, `PA1`–`PA3`, `CLEAR`, `SYSREQ`, `ATTN`

---

## Calling from PowerShell (no JSON-quoting pain)

```powershell
# GET query string (read or single-key actions)
curl.exe -sS 'http://127.0.0.1:7327/send?text=3.4&aid=ENTER'
curl.exe -sS -X POST http://127.0.0.1:7327/key/PF3

# Form-urlencoded (anything more complex than one field)
curl.exe -sS -X POST http://127.0.0.1:7327/send `
  -H 'Content-Type: application/x-www-form-urlencoded' `
  --data-urlencode 'labelLeft=Dsname Level' `
  --data-urlencode 'text=C30T158' `
  --data-urlencode 'aid=ENTER'

# Invoke-RestMethod with ConvertTo-Json (multi-step exec/login)
$body = @{ steps = @(
  @{ send = @{ fieldName='OPTION'; text='S'; aid='ENTER' } },
  @{ wait = @{ type='panel'; panelId='SDSF' } }
) } | ConvertTo-Json -Depth 5
Invoke-RestMethod http://127.0.0.1:7327/exec -Method POST `
  -ContentType 'application/json' -Body $body
```

---

## Login (legacy)

### Preferred — configured profile

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

Password lookup order: explicit body → SecretStorage `tn3270.<profile>.password` → VS Code input box.

### Fallback — manual (no profile)

```bash
# At EMSP00
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

## Common REST patterns

### ISPF navigation

```bash
# Option field is called OPTION semantically
curl.exe -sS 'http://127.0.0.1:7327/send?fieldName=OPTION&text=3.4&aid=ENTER'
curl.exe -sS -X POST http://127.0.0.1:7327/key/PF3    # back
curl.exe -sS 'http://127.0.0.1:7327/send?fieldName=OPTION&text==X&aid=ENTER'  # exit
```

### 3.4 dataset list

```bash
curl.exe -sS -X POST http://127.0.0.1:7327/send \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'labelLeft=Dsname Level' \
  --data-urlencode 'text=C30T158' \
  --data-urlencode 'aid=ENTER'
```

Scroll: `POST /key/PF8` (down), `POST /key/PF7` (up).

### SDSF

```bash
curl.exe -sS -X POST http://127.0.0.1:7327/exec -H 'Content-Type: application/json' -d '{
  "steps":[
    { "send": { "fieldName":"OPTION", "text":"S", "aid":"ENTER" } },
    { "wait": { "text":"SDSF MENU" } },
    { "send": { "fieldName":"COMMAND", "text":"DA", "aid":"ENTER" } }
  ]
}'
```

### USS via option 6

```bash
curl.exe -sS 'http://127.0.0.1:7327/send?fieldName=OPTION&text=6&aid=ENTER'
curl.exe -sS 'http://127.0.0.1:7327/send?text=OMVS&aid=ENTER'
curl.exe -sS -X POST http://127.0.0.1:7327/send \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'text=pwd' --data-urlencode 'aid=ENTER'
```

### Wait without polling

```bash
curl.exe -sS -X POST http://127.0.0.1:7327/wait -H 'Content-Type: application/json' \
  -d '{"type":"panel","panelId":"ISFPCU","timeout":10000}'

curl.exe -sS -X POST http://127.0.0.1:7327/wait -H 'Content-Type: application/json' \
  -d '{"type":"keyboardUnlock","timeout":5000}'
```

### Logoff

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

## Confidence patterns (same as MCP but curl-flavored)

### Verify panelId

```bash
curl.exe -sS http://127.0.0.1:7327/status | findstr panelId
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

Use `changedLines` directly.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `"version":"1.3.0"` | Reload VS Code window; install newer VSIX |
| `"Not connected"` | Sidebar → Connect, or `POST /connect` with `{host,port,luName?}` |
| `panelId` is null on real ISPF | Host has PANELID OFF. Run `PANELID ON` in ISPF, or match by text |
| `labelLeft` wrong field | Label too generic. Use longer surrounding text or pass row/col |
| `/login` prompts for password | Pass `"savePassword":true` once; subsequent calls use SecretStorage |
| Old script breaks on 1.4.0+ | Pass `"waitForUnlock":false` to restore 1.3.x timing |

---

## Deprecated (Bridge ≤1.3.x)

If `/status` returns no `version` field:

```bash
curl -s -X POST http://localhost:7327/send -H "Content-Type: application/json" \
  -d '{"text":"3.4","aid":"ENTER"}'
sleep 2
curl -s http://localhost:7327/screen
```

Hard-coded positions, sleeps between calls. Upgrade path: install `tn3270-bridge-2.0.0.vsix` and reload window.
