---
name: zos-sme
description: IBM z/OS System Programmer SME. Answers deep z/OS questions on Sysplex design (CFRM/SFM/ARM), DFSMS ACS routines, WLM goal-mode tuning, IPCS control-block walking, IPL/parmlib maintenance, HZSPROC Health Checker ecosystem, DB2 z/OS operator suite, RMF Monitor III interpretation, SMF record layouts, and cross-subsystem diagnosis (CICS/IMS/MQ/DB2/USS bridging). Complements the existing `mainframe` reference by focusing on SME workflows and decision trees rather than syntax lookup. Use for architecture-level questions or when the `mainframe` skill's answer is too generic.
version: 1.0.0
author: ACI PS CMM
tags: [zos, mainframe, sme, sysplex, cfrm, sfm, arm, dfsms, wlm, rmf, smf, ipcs, hzsproc, db2, cics, ims, mq, parmlib, ipl]
category: reference
---

# z/OS Senior SME — Workflow-Oriented Reference

> Complements the `mainframe` skill (which is the encyclopedia). This skill is the **runbook** — how a senior sysprog decides, diagnoses, and validates. It leans on the `mainframe-navigate` skill's `references/` folder for command-level detail.

## When to use this skill vs the `mainframe` skill

| Question style | Use |
|---|---|
| "What is the syntax of `JCL DD SPACE=`?" | `mainframe` skill |
| "Why is my batch job late? Trace root cause." | `zos-sme` |
| "List RACF classes." | `mainframe` skill |
| "Audit RACF posture for a new customer." | `zos-sme` |
| "What is a CFRM policy?" | `mainframe` skill |
| "Should I add a second CF to my sysplex?" | `zos-sme` |
| "How do I read SMF Type 30?" | `mainframe` skill |
| "Why is our capacity claim off vs the customer's baseline?" | `zos-sme` |

## Response guidelines

- **Diagnostic questions**: lead with symptoms → hypotheses → evidence needed → next steps. Follow the pattern in the `diagnose` skill.
- **Design questions**: state the trade-offs first, then the recommendation, then a validation plan.
- **Compliance / audit questions**: cite the exact RACF/DB2/HZSPROC/SMF evidence that would prove/disprove the claim. Never claim compliance without evidence.
- **Never issue mutating commands** — see `~/.claude/rules/mainframe-readonly-guardrails.md`. This rule is absolute.

---

## 1. Sysplex design & operation

### CFRM (Coupling Facility Resource Management) policy checklist

1. **Structures** — every structure declared in `CFRM` policy has:
   - `INITSIZE` — allocation at first connect
   - `SIZE` — maximum
   - `PREFLIST` — ordered CF preference (should have ≥ 2 CFs for HA)
   - `EXCLLIST` — mutually-exclusive structures (must not share CF)
   - `REBUILDPERCENT` — trigger for auto-rebuild
   - `DUPLEX(ENABLED/ALLOWED/DISABLED)` — system-managed duplexing
2. **Single points of failure**:
   - PREFLIST with only 1 CF
   - CF connected via 1 channel path
   - Sysplex CDS + alternate on same DASD LCU
   - No SFM policy (no automatic system-isolation on failure)
3. **Capacity headroom**:
   - CF storage > 2x current structure sum (rebuild + duplex)
   - CF processor utilization < 50% (headroom for rebuild + duplex)

Cross-check with SMF 74.4 (CF activity) + `D XCF,CF` + `D XCF,STR,STRNAME=name` + `D XCF,POLICY,TYPE=CFRM`.

### SFM (Sysplex Failure Management) policy checklist

1. **`ISOLATETIME`** — seconds before failed member is fenced. Too high = data corruption risk. Too low = false-positive fence during recovery.
2. **`SSUMLIMIT`** — status-update-missing threshold before triggering SFM action.
3. **`WEIGHT`** — used for tie-breaks in quorum decisions.
4. **`CONNFAIL`** — how to respond to CF connection failure.

Missing SFM policy = manual operator intervention required on failure = long RPO/RTO. Redbook citation: SG24-6488 for cross-Sysplex resiliency, SG24-6485 for parallel sysplex overview.

### ARM (Automatic Restart Manager) policy checklist

- Which address spaces are ARM-enabled? `D XCF,ARMSTATUS`.
- Restart weight (`RESTART_METHOD(SYSTERM,RETRY)`) — reasonable retry count?
- Cross-system restart eligible?
- ARM policy without SFM policy = incomplete recovery story.

### Quick decision tree — "should we add a second CF?"

```
Is there only 1 CF?
├── YES
│   ├── Any structure with PREFLIST(CF1)? → HA gap. Add CF2.
│   └── No structures — just XCF signaling? → Add CF2 for HA anyway.
└── NO
    └── CFs at > 50% utilization consistently (SMF 74.4)? → Add CF3 or upgrade.
```

---

## 2. WLM policy design

### Goal-mode fundamentals

- **Service class** — priority + goal per work unit
- **Report class** — reporting bucket (does NOT affect scheduling)
- **Classification rules** — which work → which service class (based on WGN, USER, JOBCLASS, TCLASS, PACKAGE for DB2, etc.)
- **Service coefficient** — CPU/IOC/MSO/SRB weights (rarely tuned; site-wide)
- **Application environment** — for stored proc / WAS / MQ workloads

Goals:
- **Response time (average)** — 90% completes within X seconds/ms
- **Response time (percentile)** — 90% within X (more strict)
- **Velocity** — % of dispatchable time actually running (best for batch)
- **Discretionary** — best-effort, no goal
- **System** — reserved for system started tasks (SYSSTC)

Importance: 1 (most) — 5 (least). System uses to allocate resources when goals cannot all be met.

### Diagnosis workflow — "batch is late, is WLM the cause?"

1. `D WLM,SC=<class>` → is this service class correctly defined?
2. `D WLM,POLICY` → active policy name; matches expected?
3. SMF 30 → per-job elapsed vs CPU vs delay. High elapsed but low CPU = waiting.
4. SMF 72.3 → workload activity per service class. See Delay Reasons: CPU / MPL / storage / I/O / SWAP.
5. RMF Post ▸ Workload Activity Report → same view, aggregated.
6. RMF Mon III → real-time delay analysis per address space. See DELAY report.
7. `D M=CPU` → CPU pool state. Are all CPs online? IIP/zIIP available?

If DELAY reasons show high I/O → check storage / DFSMS. High MPL → increase MAXUSERS in service class definition. High storage → paging (`D ASM`), real-storage constraint (`D M=STOR`).

Never re-tune WLM by hunches. Evidence first: SMF 72 + RMF Mon III + delay analysis.

---

## 3. DFSMS storage design

### The 4 SMS classes

| Class | Purpose |
|---|---|
| `DATACLAS` | Physical attributes (RECFM, LRECL, BLKSIZE, SPACE, VOLCNT) |
| `MGMTCLAS` | Migration/backup/retention policy (HSM control) |
| `STORCLAS` | Performance goals (cache, dual-copy, striping, dispersion) |
| `STORGRP` | Which volumes are candidates |

**ACS routines** (Automatic Class Selection) determine which class(es) get assigned. Written in ACS language (`FILTLIST`, `WHEN`, `SELECT`, `SET`). Live in `SYS1.SDSF` — no wait, in the SMS SCDS/ACDS. Editable via ISMF (option S.7 ACS Editor).

### Storage group headroom check

```
D SMS,STORGRP(ALL),LISTVOL
```

Per volume: `USED %`. Storage group full = allocation failures (`SD37 abend`).

### PDSE latch contention

`IBMPDSE.SMSPDSE1` and `SMSPDSE1` are the address spaces. Check with:
```
D OMVS,W    (waiting)
D GRS,C     (contention)
D XCF,STR,STRNAME=SYSZPDSE_STATS   (data-sharing PDSE)
```

Latch contention = long PDSE opens/closes = job elongation. Restart `SMSPDSE1` MUTATES state and requires SME + operator + change control.

### VSAM RLS

Record Level Sharing depends on `SYSIGGCAS_ECS` structure in CF. `D XCF,STR,STRNAME=SYSIGGCAS_ECS` → capacity + duplex.

---

## 4. IPCS depth beyond basics

For dump analysis basics see `~/.claude/skills/mainframe-navigate/references/ipcs-basics.md`. SME-level:

### Control-block traversal

Starting from a failing TCB, follow the chain:
```
TCB → RB chain (top RB = currently executing)
    → JSCB → JSTCB → JSTAR (job step attributes)
    → LDA (local data area) → local task pointers
    → TIOT (task I/O table) → DDNAMES + DEBs
    → JCT (via ASCB.ASCBJST) → job control table
    → ASCB.ASSB → aux storage segment block
```

Use `CBFORMAT` with the structure name — every control block has a model. Example:
```
IP CBFORMAT ASCB(X'009D2000') STRUCTURE(ASCB)
IP CBFORMAT ASSB(X'8DE0F800') STRUCTURE(ASSB)
IP LIST ASCB(X'009D2000') STRUCTURE(ASCB)  DISPLAY(ALL)
```

### LEDATA VERBEXIT depth

```
IP VERBX LEDATA 'ALL DTMH'          # verbose with data + thread + message hash
IP VERBX LEDATA 'HEAP HEAPTRACE'    # heap tracing (needs HEAPCHK)
IP VERBX LEDATA 'CB(<addr>) CEEDBG' # LE CB at specific address
```

### MTRACE (system trace)

```
IP VERBX MTRACE 'GROUP(SVC)'         # only SVC entries
IP VERBX MTRACE 'JOBNAME(EAE976B)'   # only this job
IP VERBX MTRACE 'ASID(X"0043")'      # only this ASID
IP VERBX MTRACE 'TIME(HH:MM:SS.n TO HH:MM:SS.n)'
```

Trace types: SVC, EXT (external interrupt), CLKC (clock comparator), I/O, PGM (program interrupt). RIO (real I/O), PROG (program), CPU (CPU state changes).

### Common patterns

**"Why did this job hang?"** → `IP RBSUMM` for the failing TCB → find WAIT reason → `IP LIST` the WAIT ECB address → correlate with `IP ENQ` to see if you were waiting on a dataset.

**"Which module caused the crash?"** → `IP STATUS FAILDATA` for PSW → `IP WHERE PSW` for module + offset → dig into that module.

**"Was this a compiler bug?"** → `IP LIST PSW LENGTH(64) INSTRUCTION` — disassemble around; compare to expected code.

---

## 5. HZSPROC ecosystem — SME workflow

See `~/.claude/skills/mainframe-navigate/references/health-checker.md` for commands.

### SME-level HZS workflow

1. **Site adoption baseline**: how many checks are ACTIVE? `F HZSPROC,DISPLAY,CHECKS` counts.
2. **Suppressed checks**: `F HZSPROC,DISPLAY,CHECKS,STATE=DELETED` — check who deleted and why (should be audited).
3. **Custom checks**: sites can author checks via HZSADDCK. Inventory them.
4. **PARMLIB HZSPRMxx** — how many members? `D PARMLIB` + inspect. Should match site policy.
5. **Historical trending**: `HZSPRINT LIST HISTORY` → is exception count trending up?
6. **Common exceptions to fix vs accept**: some are informational (`RSM_HVSHARE_SUPPRESS`), others are latent bombs (`XCF_SFM_ACTIVE`, `ASM_LOCAL_SLOT_USAGE`).

### Auditing an HZS-exempt environment

If a customer has HZS disabled/suppressed:
- **Missing** — every risk HZS would flag is uncalled-out. Formal risk.
- **Alternative** — customer runs OMEGAMON / MainView health checks. Verify.
- **Do not** attempt to re-enable HZS as SME — it MUTATES state. Recommend to customer with change control.

---

## 6. DB2 z/OS SME workflow

### Health check pyramid

```
Bottom (fast)   →  Top (deep)
─────────────────────────────
-DIS DB(*) SPACENAM(*) RESTRICT     [30s]
-DIS UTIL(*)                         [instant]
-DIS THREAD(*) TYPE(INDOUBT)         [seconds]
-DIS BUFFERPOOL(*) DETAIL(*)         [10s]
Catalog SELECTs (SYSTABLESPACESTATS) [minutes]
SMF 100 stats replay                 [hours]
SMF 101 accounting replay            [hours]
Trace collection (SMF 102) — MUTATES; needs approval [days]
```

Start bottom, escalate only as evidence demands.

### Common performance patterns

- **Pool too small**: `-DIS BUFFERPOOL(X) DETAIL` → SYNC READS ≫ PREFETCH READS → increase VPSIZE.
- **Log write suspend**: `-DIS LOG` → `WRITES SUSPENDED > 0` → check log dataset space, archive backlog.
- **Cluster ratio degradation**: RTS `SYSINDEXSPACESTATS.CLUSTERRATIOF` < 90% → REORG the index (needs approval).
- **Stale RUNSTATS**: `SYSTABLES.STATSTIME < NOW - 30 days` → RUNSTATS needed (mutation → approval).
- **Deadlocks**: SMF 102 IFCID 172 → capture pattern + investigate lock timeout hits.

### Data-sharing specifics

`-DIS GROUP DETAIL` → all members + IRLM + CF structure use. Every member should be at same maintenance level for zparm compatibility.

`SYSIGGCAS_ECS`, `DSNDB0G_LOCK1`, `DSNDB0G_SCA` — required CF structures for data sharing. Missing = broken sharing.

---

## 7. Cross-subsystem bridging

### CICS ↔ DB2

- CICS thread pooling for DB2: `RCT (Resource Control Table)` or `DB2CONN` (CICS TS 3.2+).
- Diagnose: `CEMT INQ DB2CONN`, `CEMT INQ DB2ENTRY`, `-DB2 DIS THREAD(*) TYPE(ACTIVE) DETAIL` filtered by `NAME(cics_appl_name)`.
- Common issue: DB2 threads exhausted → CICS queues → response time spikes.

### IMS ↔ DB2

- IMS DB2 dependent regions: `/DIS SUBSYS`, `/DIS ACTIVE REGION` → JOBNAME → correlate with `-DB2 DIS THREAD`.

### MQ ↔ CICS/DB2

- Every MQ queue manager: `DIS QMGR` for state, `DIS QSTATUS(*)` for depths, `DIS CHSTATUS(*)` for channels.
- Correlate with CICS `CEMT INQ MQCONN` and DB2 `-DIS THREAD(*) DETAIL LOCATION(*)`.

### USS ↔ MVS

- BPXBATCH bridging JCL to shell. `tsocmd/mvscmd/submit/opercmd` bridging shell to MVS.
- Filesystem ownership: MVS-visible zFS/HFS mounts under `/`. Check `D OMVS,F`.
- z/OS UNIX processes as address spaces: `D OMVS,A=ALL` → USS PID + ASID mapping.

---

## 8. IPL / Parmlib maintenance

**READ-ONLY inspection**:
- `D PARMLIB` — active concatenation.
- Read each `IEASYS00`, `LOADxx`, `LPALSTxx`, `LNKLSTxx`, `PROG` via `mainframe_read_dataset`.
- Cross-reference `D PROG,APF/LNKLST/LPA/EXIT` with what's DECLARED in parmlib. Discrepancy = dynamic change never made permanent — will disappear on next IPL.

**Never modify parmlib without SME + change control + backup**.

### Common IPL diagnostics

- `D IPLINFO` — last IPL time, LOAD parm used (LOADxx suffix), IEASYS suffix, sysres volume.
- `D IPLINFO,SYSPLEX` — sysplex context.
- `D SYMBOLS` — active resolution of `&SYSNAME.`, `&SYSCLONE.`, custom.

---

## 9. SMF record architecture (SME-relevant subset)

| Type | Content | Volume |
|---|---|---|
| 6 | JES2/JES3 printer usage | Low |
| 7 | Data-in-VIrtual (DIV) | Low |
| 14 | Non-VSAM dataset close | Very high |
| 15 | Non-VSAM dataset update close | High |
| 17 | Dataset scratch | Low |
| 18 | Dataset rename | Low |
| 21 | Tape error statistics | Low |
| 22 | System configuration | Very low (few per day) |
| 23 | SMF stats | Low |
| 30 | Job accounting (subtypes 1-5) | High |
| 42 | DFSMS storage class stats | Medium |
| 62-69 | VSAM open/close/reset | Medium |
| 70 | RMF CPU activity | Interval-based |
| 71 | RMF paging activity | Interval-based |
| 72 | RMF workload activity | Interval-based |
| 73 | RMF channel activity | Interval-based |
| 74 | RMF I/O device activity (subtypes 1..10) | Interval-based |
| 75 | RMF page dataset activity | Interval-based |
| 76 | RMF trace records | On demand |
| 77 | RMF enqueue detail | On demand |
| 78 | RMF I/O queuing | Interval-based |
| 79 | RMF Monitor II session | On demand |
| 80 | RACF security events | Event-based |
| 81 | RACF options | Low |
| 83 | RACF audit events | Event-based |
| 89 | Product usage stats | Event-based |
| 92 | z/OS UNIX file activity | High |
| 99 | WLM decisions | On demand |
| 100 | DB2 statistics | Interval-based |
| 101 | DB2 accounting | Per-thread-end |
| 102 | DB2 performance/audit | On demand |
| 110 | CICS statistics | Interval + task-end |
| 113 | Hardware counters (HIS) | Interval-based |
| 115 | MQ statistics | Interval-based |
| 116 | MQ accounting | Event-based |
| 117 | IEHINITT tape init | Low |
| 118 | TCP/IP statistics | Interval-based |
| 119 | TCP/IP configuration/statistics | Interval + event |
| 120 | WebSphere Application Server | Event-based |

### Subtype watchpoints

- **SMF 30 subtype 4** = job-level totals. Subtype 5 = step-level. **Never sum both** (see rule 10 of `mainframe-readonly-guardrails.md`).
- **SMF 42 subtype 6** = DFSMS media report; subtype 15/16 = zFS activity.
- **SMF 74 subtype 4** = XCF (coupling facility) activity; subtype 5 = cache subsystem; subtype 8 = ESS/SSD activity.
- **SMF 80 subtype** = RACF event type (login fail, dataset access, etc.).
- **SMF 100 IFCIDs** = the DB2 statistics categories (bufferpool = 3, log = 4, EDM = 202, etc.).
- **SMF 113 subtype 1** = HIS (Hardware Instrumentation Services) — CPI, MIPS per LPAR when HIS is enabled.

---

## 10. Decision tree — "customer batch is late"

```
1. When did it start being late? (correlate with IPL / policy change / capacity change)
2. Which job(s) specifically? (SMF 30 filtered)
3. Elapsed vs CPU — is it CPU-bound or waiting?
   ├── CPU-bound → is CPU headroom available? (RMF 70)
   │   ├── Yes → why isn't job dispatching? (WLM classification wrong? SMF 30 delay reasons.)
   │   └── No → capacity issue. Upgrade or reschedule.
   └── Waiting → what for?
       ├── I/O → RMF 74 device activity + DFSMS
       ├── Storage → paging (D ASM) or real (D M=STOR)
       ├── ENQ → D GRS,C, SDSF ENQ
       ├── DB2 lock → -DIS THREAD(*) DETAIL + SMF 100
       ├── CF service → SMF 74.4
       └── Other subsystem (CICS, MQ, IMS) → their own diagnostics
4. Is this job on the TWS critical path? Late here = late everywhere downstream.
5. Recent changes? Change management review.
```

Every step above is READ-ONLY. Never `C jobname`, `FORCE`, or `RUNSTATS` a live production DB2 table as part of diagnosis.

---

## 11. Escalation to SHC or specialized skills

- **Coletor-driven audit**: point the customer at the SHC (Onda A+ = 1.5.0+). The `sysinfo.txt` + `phase_smf` + `phase_db2cat` + `phase_rmf` + `phase_healthchk` (Onda B) + `phase_sdsf` (Onda B) covers most SME questions with evidence, not opinion.
- **Product-specific**: acquirer, base24, interchange, issuer, postilion, prm, asset — dedicated agents. Do not answer product-specific questions from generic z/OS knowledge.
- **Diagnosis flow**: `mainframe-diagnose` skill for structured post-mortem.

---

## Related files

- `~/.claude/skills/mainframe/SKILL.md` — z/OS encyclopedia (13 IBM Redbooks compiled)
- `~/.claude/skills/mainframe-diagnose/SKILL.md` — diagnosis flow
- `~/.claude/skills/mainframe-navigate/SKILL.md` — TN3270 navigation entry point
- `~/.claude/skills/mainframe-navigate/references/*.md` — 12 command-level reference files
- `~/.claude/rules/mainframe-readonly-guardrails.md` — non-negotiable safety rules
