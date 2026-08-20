# SDSF Depth Reference

SDSF (System Display and Search Facility) is the primary window into JES2/JES3, the syslog, RMF Monitor III, and MVS console interaction on z/OS. Everything below assumes you are inside SDSF (ISPF Option S). All patterns are read-only unless flagged.

---

## Panels

| Cmd | Panel | Shows |
|---|---|---|
| `DA` | Display Active | Active batch + TSO + STCs (started tasks) |
| `ST` | Status | All jobs in JES (active + input queue + output queue) |
| `I` | Input queue | Jobs waiting to run |
| `H` | Held output | Jobs whose output is held (HOLD=YES or class held) |
| `O` | Output queue | Jobs whose output is available |
| `LOG` | System log | SYSLOG (JES2) or OPERLOG (log stream) — merged console messages |
| `SR` | System requests | Outstanding WTORs (replies needed) |
| `ULOG` | User log | Console commands YOU issue via the `/` prefix |
| `ENQ` | Enqueues | GRS/GRS-lite dataset/resource contention |
| `JC` | Job classes | JES2 class definitions + initiator status |
| `INIT` | Initiators | Batch initiator status |
| `PR` | Printers | JES printers |
| `LI` | Lines | JES lines |
| `NO` | Nodes | JES NJE nodes |
| `SO` | Spool offload | JES spool offload |
| `SP` | Spool volumes | JES spool utilization per volume |
| `RES` | Resources | JES resource utilization |
| `RM` | Resource monitor | RMF-lite (Mon III via SDSF, if RMF DDS is running) |
| `PS` | Processes | USS processes (`ps` equivalent from ISPF) |
| `WORK` | Work | Consolidated job/task view |
| `CK` | Health Checker | IBM Health Checker exceptions (if `HZSPROC` running) |

---

## Filters (SDSF primary command line)

Filters persist for the session. Set them BEFORE entering a panel to narrow scope.

```
PREFIX EAE*              Jobs whose jobname starts with EAE
OWNER C30T158            Jobs owned by that TSO userid
DEST LOCAL               Output destination
SYSNAME SPS1             Jobs on this LPAR (sysplex-wide when unset)
JCLASS A                 Batch job class
OCLASS X                 Output class
GROUP GRPACQ             Job group
FILTER JNM EQ EAE976B    Ad-hoc column filter (JNM=jobname, ANY column)
FILTER *                 Clear all filters
```

Common combos:
```
PREFIX EA*
OWNER *
DEST *
SYSNAME *
```

...gives you every EA* job across the sysplex regardless of owner.

---

## Line commands on a job row

Position cursor on the row and type in the leftmost cell:

| Cmd | Effect |
|---|---|
| `S` | Select job → shows DDNAMES list |
| `SB` | Select and browse (DDNAMEs into ISPF Browse) |
| `SE` | Select and edit (careful — only for JCL rewriting a submittable copy) |
| `?` | Show job DDNAME list (same as `S` but explicit) |
| `X` | Print (send output to spool print class) |
| `XD` | Export to dataset (opens dialog) |
| `XDC` | Export as CSV (SDSF-friendly for automation) |
| `A` | Release (from HOLD) — MUTATION |
| `H` | Hold — MUTATION |
| `P` | Purge — MUTATION (drops the job's spool) |
| `C` | Cancel running job — MUTATION |
| `CD` | Cancel with dump — MUTATION |
| `E` | Requeue for execution — MUTATION |

**Read-only rule**: `S/SB/?/XDC` are safe. `A/H/P/C/CD/E` mutate — need explicit user confirmation.

---

## Drilling into a job's SYSOUT

```
mainframe_send fieldName=COMMAND text="ST" aid=ENTER
mainframe_send fieldName=COMMAND text="PREFIX EAE976B" aid=ENTER
mainframe_send fieldName=COMMAND text="OWNER *" aid=ENTER
mainframe_send fieldName=COMMAND text="ST" aid=ENTER     # refresh
# Position cursor on the target row and type S in col 1
mainframe_cursor row=<jobrow> col=1
mainframe_send text="S" aid=ENTER
```

You are now on the DDNAME list. Common DDNAMEs:

| DDNAME | Content |
|---|---|
| `JESMSGLG` | JES2 messages (submit, allocate, IEF codes, RC) |
| `JESJCL` | Expanded JCL (INCLUDE resolved, SET substituted, JOBLIB/STEPLIB visible) |
| `JESYSMSG` | JES2 system messages (dataset allocation, catalog, ABEND) |
| `SYSPRINT` | Application stdout from each step |
| `SYSTSPRT` | TSO/E stdout (from IKJEFT01 batch TSO) |
| `SYSOUT` | Application stdout (per-step or step-specific) |
| `CEEDUMP` | LE (Language Environment) dump — ABEND stack |
| `SYSUDUMP` | User dump (S0Cx, S878, S806, etc.) |
| `SYSMDUMP` | Machine-readable dump (feed to IPCS) |
| `SYSABEND` | Formatted ABEND dump |
| `SNAPOUT` | Application SNAP (debug snapshot) |
| `DDLIST` | z/OS UNIX process output (BPXBATCH) |

Select `S` next to the DDNAME to browse. Or `SB` to open in ISPF Browse for FIND.

---

## Exporting SYSOUT to a dataset (for programmatic parsing)

```
# On the DDNAME row, type XDC (export to CSV dataset)
mainframe_cursor row=<ddrow> col=1
mainframe_send text="XDC" aid=ENTER
# XDC dialog appears — enter target dataset
mainframe_send labelLeft="Dataset name" text="C30T158.EXPORT.JESMSGLG" aid=ENTER
```

Result: a sequential dataset you can `mainframe_read_dataset` for automation.

---

## SYSLOG / OPERLOG (`LOG`)

```
mainframe_send fieldName=COMMAND text="LOG" aid=ENTER
# Set time window
mainframe_send fieldName=COMMAND text="LOG O" aid=ENTER    # switch to OPERLOG (log stream)
mainframe_send fieldName=COMMAND text="TIME 06:00 12:00" aid=ENTER
mainframe_send fieldName=COMMAND text="FIND 'IEE041I'" aid=ENTER
mainframe_send fieldName=COMMAND text="FIND 'IEA911E'" aid=ENTER    # SVC dump captured
```

Useful FIND patterns:
- `IEA911E` — SVC dump captured
- `IEF450I` — job/step ABEND
- `IEF404I` — job ended
- `IEF196I` — allocation issue (dataset)
- `ICH408I` — RACF violation
- `IEC030I` / `IEC031I` — I/O error
- `IEA995I` — MVS symptom dump
- `IEE143I` — WTOR being issued

---

## ULOG — issue console commands from SDSF

`ULOG` opens your personal console log. Type MVS commands prefixed with `/`:

```
mainframe_send fieldName=COMMAND text="ULOG" aid=ENTER
mainframe_send fieldName=COMMAND text="/D T" aid=ENTER          # time of day
mainframe_send fieldName=COMMAND text="/D IPLINFO" aid=ENTER    # IPL info
mainframe_send fieldName=COMMAND text="/D M=CPU" aid=ENTER
mainframe_send fieldName=COMMAND text="/D XCF,STR" aid=ENTER
```

Requires `OPERCMDS` MCSOPER READ (site-specific). If denied, response contains `IEE345I ULOG NOT AUTHORIZED`.

**Mutation guardrail**: `/C jobname`, `/FORCE`, `/V device,OFFLINE`, `/SETPROG APF,…`, `/SET SMF=xx`, `/Z EOD` — never issue without explicit user confirmation. See `console-commands.md` for full blocklist.

---

## ENQ — contention

```
mainframe_send fieldName=COMMAND text="ENQ" aid=ENTER
mainframe_send fieldName=COMMAND text="FILTER QNAME EQ SYSDSN" aid=ENTER  # dataset ENQs
```

Shows jobs holding vs waiting on QNAME/RNAME pairs. Classic culprit of "job stuck": a dataset ENQ held by another job with `E` (exclusive) while yours needs `S` (shared or exclusive).

---

## Health Checker (`CK`)

```
mainframe_send fieldName=COMMAND text="CK" aid=ENTER
mainframe_send fieldName=COMMAND text="SORT STATE D" aid=ENTER    # exceptions first
```

Line commands on a check row:
- `S` — show check history
- `A` — activate (mutation)
- `D` — deactivate (mutation)
- `R` — refresh interval (mutation)

For batch export: `HZSPRINT` utility (see `health-checker.md`).

---

## SDSF via programmatic REXX (for SHC-style automation)

When you need to script SDSF from JCL (not TN3270), use REXX with the `ISFEXEC` host command:

```rexx
/* REXX */
address ISFEXEC "ST"
say "isfrows=" isfrows
do i = 1 to isfrows.0
  say jname.i job#.i status.i
end
```

Handy stems: `jname.`, `job#.`, `owner.`, `class.`, `sysname.`, `cpu.`, `queue.`, `retcode.`.

Requires `SDSF` class READ profile access (`ISFCMD.ODSP.STATUS.jesx`, `ISFCMD.DSP.ACTIVE.jesx`).

---

## Pitfalls

- **`panelId` after each hop.** After `ULOG` you go to `ISFULOG`. After `S` on a job you go to `ISFPCU`. `PF3` gets back. Verify with `mainframe_status`.
- **PREFIX persists.** Setting `PREFIX EA*` then switching to `LOG` keeps the prefix; irrelevant for LOG but you may be surprised on the next `ST`.
- **XDC dataset must not exist.** SDSF refuses to overwrite. Prefix your export dataset with a timestamp.
- **ULOG is your log**, not the system log — use `LOG` for OPERLOG. ULOG only shows commands YOU issued this session.
- **RMF Mon III via `RM` requires RMF DDS running** on this LPAR. If `RM` says "RMF NOT AVAILABLE", use ERBSMFI or GPMSERVE from JCL (see the SHC `phase_rmfmon3`).
