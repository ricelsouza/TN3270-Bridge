# MCP Tools Reference — `mcp__tn3270-bridge__mainframe_*`

TN3270 Bridge 2.0.0+ exposes 15 typed MCP tools that supersede the raw REST API on port 7327. Prefer these — they are serialized per session, they surface errors as tool results (not stderr), and they encode the same idioms this skill teaches. If the tools are absent in your session, fall back to `curl` patterns in the parent `SKILL.md`.

Every tool below shows: signature, semantics, when to use, and a minimum-viable call.

---

## Connection & session

### `mainframe_status`
No arguments. Returns bridge + connection + screen state.

```json
{
  "bridge": "running",
  "version": "2.2.0",
  "connected": true,
  "connection": {"host":"mfz900acw2", "port":23, "tls":false},
  "screen": {"rows":24, "cols":80, "cursorRow":4, "cursorCol":14, "fieldCount":119, "keyboardLocked":false, "panelId":"ISR@PRI"}
}
```

Use as a **first call every session** to confirm you are where you think you are. `panelId: null` means you are on a non-ISPF panel (EMSP00 banner, TSO READY, or a broken hop).

### `mainframe_connect`
Args: `host` (required), `port=23`, `luName?`, `tls?`, `model=model2`.

Replaces the default session. For TLS on port 992: `tls=true`. Advanced TLS knobs (CA bundle, min version, cipher, hostname override) go via VS Code `tn3270Bridge.tls.*` settings — the tool call itself only takes `tls: true|false`.

Errors surface as `errorCode`:
- `TLS_SELF_SIGNED` / `TLS_CA_NOT_TRUSTED` — set `caBundle` or `rejectUnauthorized:false` for dev
- `TLS_CERT_EXPIRED` — renew or bypass for testing
- `TLS_HOSTNAME_MISMATCH` — set `tls.sni` in settings
- `TLS_VERSION_MISMATCH` — wrong port; try 23 for plain or lower `tls.minVersion`
- `CONN_REFUSED` / `CONN_RESET` / `CONN_TIMEOUT` — wrong host, firewall, or plain/TLS mismatch

### `mainframe_disconnect`
Args: `session?`. Disconnects without touching host datasets/jobs. Default session requires `sessionId="default"` explicitly.

### `mainframe_sessions`
No arguments. Lists default + named sessions with their state.

### `mainframe_session_create`
Args: `host` (required), `port=23`, `luName?`, `tls?`, `model=model2`, `sessionId?`.

Opens an ADDITIONAL session without replacing default. Useful for parallel LPAR work or for holding a long-running SDSF ULOG channel while another session drives ISPF.

### `mainframe_session_delete`
Args: `sessionId` (required). Removes a named session.

### `mainframe_login`
Args: `profile` (required), `userid` (required).

Runs a VS Code `tn3270Bridge.loginProfiles.<profile>` script. **Password comes from SecretStorage** — never pass it in the call. Profiles configure the exact keystroke sequence per site.

---

## Screen reading

### `mainframe_read_screen`
Args: `nonblank=true`, `session?`.

Returns compact lines + `cursor` + `panelId` + `keyboardLocked`. `nonblank=true` (default) strips empty rows and cuts tokens ~70% vs the full 24×80 dump. Hidden fields (passwords) are always masked.

### `mainframe_fields`
Args: `session?`.

Returns each input field with position, length, protected flag, hidden flag, and its resolved label (from `labelLeft`/`labelAbove`). Use when you need to place text semantically instead of by row/col.

### `mainframe_find`
Args: `text` (required, case-insensitive), `session?`.

Locates text on the current screen and returns row/col. Cheap way to verify a marker before drilling.

### `mainframe_history`
Args: `last=5` (max 10), `session?`.

Returns up to 10 recent masked screen snapshots. Use for retroactive diagnosis when navigation went sideways.

### `mainframe_table`
Args: `headerRow?`, `session?`.

Parses the current columnar screen (SDSF DA, ISPF 3.4 dataset list, RMF Mon III workload list) into structured `{headers: [...], rows: [{...}]}`. Beats regexing raw text.

---

## Input & navigation

### `mainframe_cursor`
Args: `row?`, `col?`, `field?`, `fieldName?`, `labelLeft?`, `labelAbove?`, `session?`.

Moves the local cursor WITHOUT pressing any AID key. The cursor position is embedded in every buffer sent to the host — repositioning locally is enough for unformatted fields (SDSF selection, ISPF Edit, raw OMVS). No screen redraw.

### `mainframe_send`
Args: `text?`, `aid="ENTER"`, `clearField=true`, `row?`, `col?`, `field?`, `fieldName?`, `labelLeft?`, `labelAbove?`, `session?`.

Types text into a resolved field and presses an AID key. Field resolution priority: `fieldName` (semantic like `OPTION`, `COMMAND`, `CURSOR`) → `labelLeft` → `labelAbove` → `field` (index) → `row`+`col`. Response includes `panelId`, `panelChanged`, `changedLines`, `ispfMessage.{short,long}`, `keyboardLocked`.

Waits for keyboard unlock by default — the returned screen is the post-AID state. Pass `waitForUnlock:false` for fire-and-forget (rare).

AID keys: `ENTER`, `PF1`–`PF24`, `PA1`–`PA3`, `CLEAR`, `SYSREQ`, `ATTN`.

### `mainframe_wait`
Args: `type="text"`, `text?`, `panelId?`, `timeout=10000`, `session?`.

Types: `text` (wait for substring), `panel` (wait for `panelId==X`), `keyboardUnlock`, `screenChange`. Returns `{found: bool, type, screen: [...]}`. Beats sleeping — returns the instant the condition is real.

### `mainframe_exec`
Args: `steps: [...]`, `stopOnError=true`, `session?`.

Chains send/wait steps atomically inside one serialized call. Each step is `{send: {...}}` or `{wait: {...}}`. Great for known-good navigation sequences (login flow, ISPF hop chains, SDSF drill).

### `mainframe_interrupt`
Args: `session?`.

Sends Telnet Interrupt Process. Use ONLY when keyboard is confirmed locked AND you know why (hung TSO command, runaway REXX). Do not use as generic "unstuck" — reason first.

---

## Multi-page / dataset reads

### `mainframe_read_all`
Args: `scrollKey="PF8"`, `maxPages=200`, `stopText="BOTTOM OF DATA"`, `timeout=30000`, `skipHeaderRows=0`, `session?`.

Accumulates lines across screens with an explicit stop reason. Handles OMVS `MORE...`, SDSF long output, ISPF Browse, IPCS output. Cleaner than hand-rolling PF8/PF10 loops.

Common stop texts:
- ISPF Browse: `"BOTTOM OF DATA"`
- SDSF O output: `"BOTTOM OF DATA"`
- OMVS: use `stopText="$"` and low `maxPages` (OMVS is line-mode, not page-mode)
- ISPF Edit: `"BOTTOM OF DATA"`

### `mainframe_read_dataset`
Args: `dsn` (required), `member?`, `session?`.

Reads a sequential dataset or PDS member through ISPF View and restores the prior panel. Capped at 256 KiB. **Read-only**. Ideal for `SYS1.PARMLIB`, DB2 zparm dumps, JCL PDS members.

---

## Job submission

### `mainframe_submit_job`
Args: `dsn` (required), `member?`, `session?`.

Runs `TSO SUBMIT '<dsn>[(<member>)]'` from an ISPF command line. Returns the resulting screen — parse the JOBID from `IKJ56250I JOB <jobname>(JOBnnnnn) SUBMITTED`. Poll `SDSF ST` afterwards for RC.

**Guardrail**: this can execute arbitrary JCL. Read the member first (`mainframe_read_dataset`) if you did not author it. Anything mutating (DELETE, DROP, UPDATE, INSERT, IEBGENER DISP=CATLG over non-owned HLQ, RACF PE, SETPROG APF) requires explicit user confirmation.

For BUILD → SUBMIT → POLL → FETCH-SPOOL → PARSE-RC end-to-end, see [`jcl-submit-loop.md`](jcl-submit-loop.md).

### `mainframe_write_dataset`
Args: `dsn` (required), `content` (required, string or array), `member?`, `session?`.

Replaces a sequential dataset or PDS member through ISPF Edit. Incomplete writes are CANCELLED before SAVE (safety). **Guardrail**: only write under HLQs the user confirmed as owned by the current userid. Never write to `SYS1.*`, `RACF.*`, `DB2.*`, `HZS.*`, or any production dataset without explicit approval.

---

## Cheat sheet: which tool for which need

| Need | Tool |
|---|---|
| Check where I am | `mainframe_status` |
| Log in | `mainframe_login` |
| See the current screen | `mainframe_read_screen` |
| See input fields with labels | `mainframe_fields` |
| Find a marker on screen | `mainframe_find` |
| Move cursor without pressing key | `mainframe_cursor` |
| Type + press key | `mainframe_send` |
| Wait for text/panel/unlock/change | `mainframe_wait` |
| Chain steps atomically | `mainframe_exec` |
| Read multi-page output | `mainframe_read_all` |
| Read a PS/PDS member | `mainframe_read_dataset` |
| Parse a tabular screen | `mainframe_table` |
| Submit a JCL member | `mainframe_submit_job` |
| Write a PS/PDS member (guarded) | `mainframe_write_dataset` |
| Break a hung TSO | `mainframe_interrupt` |
| Retro-diagnose a bad hop | `mainframe_history` |
| Open a second parallel session | `mainframe_session_create` |
