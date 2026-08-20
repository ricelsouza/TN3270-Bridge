# Mainframe Read-Only Guardrails — Non-Negotiable

**Applies to**: every session that touches a z/OS mainframe (via `mainframe-navigate` skill, `zos-sme` agent, `mcp__tn3270-bridge__mainframe_*` tools, `mainframe-diagnose`, `mainframe-health`, or any SHC workflow).

**Default posture**: read-only. Every operation is presumed to be read-only unless the user explicitly requests a mutation AND you have shown them the exact command/JCL AND they confirmed.

**Rule of thumb**: if you cannot tell whether a command reads or mutates state, treat it as mutation and confirm.

---

## Rule 1 — Never issue RACF mutations without explicit confirmation

Commands NEVER issued without user confirmation:

- `PE` / `PERMIT` — grant/revoke resource access
- `AU` / `ADDUSER` — create user
- `ALU` / `ALTUSER` — modify user
- `DU` / `DELUSER` — remove user
- `AG` / `ADDGROUP` — create group
- `ALG` / `ALTGROUP` — modify group
- `DG` / `DELGROUP` — remove group
- `CO` / `CONNECT` — add user to group
- `RE` / `REMOVE` — remove user from group
- `RDEF` / `RDEFINE` — create/modify resource profile
- `RDEL` / `RDELETE` — delete resource profile
- `RALT` / `RALTER` — alter resource profile
- `ADDSD` / `ALTDSD` / `DELDSD` — dataset profile mutations
- `SETR` / `SETROPTS` — global settings
- `PASSWORD` / `PHRASE` mutations
- `PWSYNC` — RRSF password propagation
- `RVARY` — deactivate RACF (extreme)

Safe alternatives (READ-ONLY): `LU`, `LG`, `LISTDSD`, `RLIST`, `SEARCH`, `LISTUSER`, `LISTGRP`, `RACDCERT LIST`. See [`~/.claude/skills/mainframe-navigate/references/racf-inspect.md`](../skills/mainframe-navigate/references/racf-inspect.md).

---

## Rule 2 — Never submit destructive JCL without confirmation

Before ANY `mainframe_submit_job` or `SUBMIT`, review the JCL for these patterns and confirm if present:

- **IDCAMS**: `DELETE`, `DELETE ... PURGE`, `DELETE ... ERASE`, `ALTER ADDVOLUMES/REMOVEVOLUMES`, `DEFINE ... REPLACE`, `EXPORT/IMPORT` on production VSAM
- **IEBGENER / IEBCOPY**: `DISP=(NEW,CATLG)` on target overwriting existing catalog entry (check first with `LISTC ENT('dsn')`)
- **DB2 utilities**: `RUNSTATS FORCE`, `REORG`, `LOAD REPLACE`, `RECOVER`, `MODIFY RECOVERY`, `COPY` targeting production
- **DB2 SQL via DSNTEP2**: `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, `ALTER`, `GRANT`, `REVOKE`, `TRUNCATE`, `RENAME`
- **DFSMShsm**: `HRECALL`, `HDELETE`, `HMIGRATE`, `HBACKDS` when moving material data
- **DFSMSdss**: `RESTORE`, `COPY REPLACE` on production
- **BPXBATCH**: shell commands invoking `rm -rf`, `dd`, `cp` over existing files, `chmod 777`, or `su`
- **PROGRAM=BPXBATSL PARM='PGM /path/to/binary'** — verify the binary
- **Any custom utility with a name you don't recognize** — read it via `mainframe_read_dataset` first

Rule when uncertain: **read the JCL member first** with `mainframe_read_dataset`, show it to the user, ask for confirmation.

---

## Rule 3 — Never issue mutating console commands without confirmation

Commands NEVER issued without user confirmation (this is the blocklist from [`console-commands.md`](../skills/mainframe-navigate/references/console-commands.md), summarized):

### Job/task control
- `C jobname` / `C jobname,DUMP` — CANCEL
- `P jobname` — STOP
- `FORCE jobname,ARM` / `FORCE jobname,QUICK` — force termination
- `S jobname` / `S procname` — START

### Device / system state
- `V device,OFFLINE` / `V device,ONLINE`
- `V PATH,online/offline`
- `V NET,INACT/ACT`
- `V xcf,SYSNAME,OFFLINE`
- `Z EOD` (quiesce)
- `Z NET,QUICK`
- `Z ABEND`

### Parmlib / system config
- `SETPROG APF,ADD|DELETE`
- `SETPROG LNKLST,DEFINE|UNDEFINE|ACTIVATE|DEACTIVATE`
- `SETPROG LPA,ADD|DELETE`
- `SETPROG EXIT,ADD|DELETE|MODIFY`
- `SET IEA=xx` / `SET GRS=xx` / `SET SMF=xx` / `SET IOS=xx` / `SET OMVS=xx` / `SET SMS=xx` / `SET LOGR=xx` / `SET WLM=xx` / `SET ALLOC=xx` / `SET DEVSUP=xx`
- `SETSMF SYS(TYPE(...))`
- `SS TYPE=n`

### XCF / Coupling Facility
- `SETXCF START|STOP,POLICY=name`
- `SETXCF STOP,STR=name`
- `SETXCF FORCE,STR=name,CON=...`
- `SETXCF PRSMPOLICY`
- `SETXCF START,REBUILD/ALTER`

### OMVS
- `F BPXOINIT,SHUTDOWN=FORKS`
- `F BPXOINIT,SUPERKILL=pid`
- `SETOMVS RESET=(xx)`

### Slip traps / trace
- `SLIP SET/MOD/DEL/DISABLE/ENABLE/RESET`
- Any modifying `TRACE` verb

### Dumps
- `DUMP TITLE=(...)` (allocates DASD)
- `DUMPDS ADD/DEL`
- `CHNGDUMP SET`

Safe alternatives (READ-ONLY): `D` (DISPLAY) verbs cover 99% of inspection needs. See [`console-commands.md`](../skills/mainframe-navigate/references/console-commands.md).

---

## Rule 4 — Only write to HLQs the user confirmed as owned

Applies to `mainframe_write_dataset`, `submit_job` targets, `IDCAMS DEFINE`, `IEBCOPY OUTDD`, `cp -T "//'PDS(MBR)'"`.

**Allowed by default** (no explicit confirmation needed for creation, but confirmation needed for overwrite):
- `<userid>.**` where `<userid>` matches the current TSO/logon userid
- Datasets explicitly listed in `OUT_DIR`, `HLQ`, `REXX_PDS`, `JCL_PDS`, `JCLTMP` config of an active SHC run

**Never write without explicit user confirmation**:
- `SYS1.**`, `SYS2.**`, `SYS3.**` — system datasets
- `SYSCTLG.**`, `SYSVTOC.**`
- `RACF.**` — RACF database (never modifiable this way anyway)
- `DB2.**`, `DSN*.**`, `<SSID>.**` — DB2 subsystem datasets
- `HZS.HZSPDATA` — Health Checker data
- `TWS.**`, `EQQ**.**` — TWS/OPC datasets
- `HSM.**`, `MHLQ.**` — HSM datasets
- Any dataset under a HLQ marked as a production HLQ by the customer
- Any GDG generation `(+1)` when the base or model belongs to a shared/prod HLQ

If a dataset name pattern is ambiguous (e.g. `<userid>.PROD.**` where user runs under a prod TSO id), STOP and ask.

---

## Rule 5 — Every collection defaults to read-only

When implementing an SHC phase or an SME diagnostic:
- Default to **READ operations** unless a mutation is required by the SOW and pre-approved.
- If a phase can be implemented via a `D` (DISPLAY) command instead of a modifying command, use the `D` variant.
- If a phase is truly ambiguous (both reads and writes state), split it: read phase first, mutation phase gated by explicit user confirmation.
- Documented AUTH_DENIED artifact is preferred over an unsafe workaround. Denial from RACF is not a challenge to bypass.

Preserve the 5 SHC evidence states: **OK / EMPTY / INCONCLUSIVE / AUTH_DENIED / FAILED**. Missing evidence stays `INCONCLUSIVE`; do not classify silently as `OK`.

---

## Rule 6 — Before any submit that is not clearly SELECT/DISPLAY, show the JCL

Standard pattern:

1. Build the JCL.
2. Read it back to the user in a fenced code block.
3. State plainly: "This JCL will do X — it {is / is not} read-only. Confirm to submit."
4. Only after explicit "yes/confirm/go" from the user, invoke `mainframe_submit_job` or `SUBMIT`.

Skip the confirmation ONLY when:
- The JCL executes `IEFBR14` (no-op program) with `DUMMY` DDs.
- The JCL runs `IDCAMS LISTCAT/PRINT` only — no `DELETE/DEFINE/ALTER/REPRO`.
- The JCL runs `DSNTEP2` with SQL containing only `SELECT` statements (verified by grep).
- The JCL runs `HZSPRNT LIST EXCEPTIONS` — pure read.
- The JCL runs `IFASMFDP OPTIONS(DUMP)` reading INDD and writing to a private OUTDD under the user's HLQ. Even here, mention the target.

Never skip when the JCL includes any of: `IEBGENER`, `IEBCOPY`, `IDCAMS DELETE|DEFINE|ALTER|REPRO`, `BPXBATCH`, `IKJEFT01` (batch TSO — can do anything), or a custom program name.

---

## Rule 7 — Diagnostic commands stay diagnostic

`mainframe_interrupt` (Telnet IP) sends an actual interrupt to the host — it CAN kill a running TSO command. Use ONLY when:
- Keyboard is verifiably locked (`mainframe_status` shows `keyboardLocked: true`).
- You know why (identified hung command, runaway REXX).
- User has been informed and consented, OR the interrupt is the only path forward AND you say so.

Never issue `interrupt` as a generic "unstuck" reflex.

`mainframe_disconnect` — safe (just drops session). But confirm before doing it on a session the user was actively using.

---

## Rule 8 — Bash / PowerShell tools do not touch the mainframe

The mainframe is reached ONLY through:
- `mcp__tn3270-bridge__mainframe_*` tools (preferred)
- Legacy REST API on `http://localhost:7327` via `curl` (fallback)

**Do NOT**:
- Use the Bash tool to invoke `ssh`, `sftp`, or `ftp` to production mainframe hosts without user confirmation and correct security context.
- Store passwords in files. Login profiles use SecretStorage.
- Attempt `tsocmd` or `mvscmd` from PowerShell on the workstation — those are z/OS USS binaries.
- Execute shell scripts on the mainframe from PowerShell — go through the bridge.

The FTP helper for SHC deployment uses agreed customer credentials and stays in scope of the coletor delivery. That is the ONLY sanctioned FTP path.

---

## Rule 9 — Never declare parsers golden-qualified without SME approval

Applies to SHC binary parsers: SMF 30 (job accounting), RMF 70‑74, DB2 SMF 100/101, and any future SMF 14/15/42/80/89/113/118/119/120 parsers.

Golden qualification requires ALL of:
1. Raw input (real SMF from a controlled LPAR).
2. Independent expected output (from IBM Interpret, MXG, RMF Postprocessor, or another SME-owned oracle).
3. Manifest with dates, LPAR, SMFPRMxx settings, SME approver name+email+date.
4. 100% match on `qualify_parser.py`.
5. SHA-256 checksums of raw + expected on file.

Until all 5 are satisfied, the parser is labeled `engineering-preview` in the report and its output is treated as heuristic, not authoritative.

---

## Rule 10 — SMF 30 subtype 4 and 5 are different records — never conflated

- **Subtype 4** = job-level totals (one record per job at end)
- **Subtype 5** = step-level records (one per step execution)

Never sum both when reporting "total CPU used by job X." Subtype 5 elapsed/CPU already rolled up into subtype 4. Doubling this is a hard architectural error and produces inflated capacity claims.

---

## Escalation path

If a user asks for a mutation this rule blocks:

1. State clearly what the mutation would do.
2. State what RACF resource / authorization would gate it.
3. Offer a read-only equivalent if one exists.
4. If the user still wants the mutation, remind them of this rule, ask for explicit confirmation phrase (`Yes, proceed with the mutation X on target Y`), and only then invoke.

If a user challenges this rule ("this is my sandbox, just run it"): respect that autonomy if the user is confirmed to be operating on a personal/dev/test environment they own. Still confirm the exact command.

---

## Version

This rule is version **1.0** (2026-07-28). Update when new mutation classes are identified.

## Cross-references

- [`~/.claude/skills/mainframe-navigate/SKILL.md`](../skills/mainframe-navigate/SKILL.md) — navigation entry point
- [`~/.claude/skills/mainframe-navigate/references/racf-inspect.md`](../skills/mainframe-navigate/references/racf-inspect.md) — read-only RACF patterns
- [`~/.claude/skills/mainframe-navigate/references/db2-operator.md`](../skills/mainframe-navigate/references/db2-operator.md) — read-only DB2 patterns
- [`~/.claude/skills/mainframe-navigate/references/console-commands.md`](../skills/mainframe-navigate/references/console-commands.md) — full DISPLAY matrix + mutation blocklist
- SHC evidence contract: `EVIDENCE_CONTRACT_MODE=ENFORCE` in coletor config
