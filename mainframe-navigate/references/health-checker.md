# IBM Health Checker for z/OS Reference

The IBM Health Checker (HZSPROC) is z/OS's built-in continuous validation engine. It runs 200+ checks (IBM-supplied + optional site-authored) against system configuration and reports exceptions. This is the closest thing z/OS has to a `hz` doctor — and it is criminally underused.

---

## Address space + datasets

- **HZSPROC** — the started task. Confirm running: `D A,HZSPROC` (or SDSF `DA`, `PREFIX HZSPROC`).
- **HZS.HZSPDATA** — persistent data VSAM (check results across IPLs). Requires READ (site profile) to inspect.
- **HZSPRINT** — batch utility to print check results to SYSPRINT.
- **HZS OPERCMDS** — MODIFY commands to the started task.

RACF profiles typically involved:
- `FACILITY HZS.**` — inspect/modify checks
- `DATASET HZS.HZSPDATA` — READ for HZSPRINT batch
- `OPERCMDS MVS.MCSOPER.HZS.*` — issue F HZSPROC commands
- `LOGSTRM SYSPLEX.OPERLOG` — read consolidated log

---

## MODIFY HZSPROC — operator commands

### Read-only DISPLAY variants

```
F HZSPROC,DISPLAY,CHECKS                          # all checks + state
F HZSPROC,DISPLAY,CHECKS,STATE=EXCEPTION          # only exceptions
F HZSPROC,DISPLAY,CHECKS,STATE=EXCEPTION-HIGH     # high-severity only
F HZSPROC,DISPLAY,CHECKS,STATE=EXCEPTION-MED
F HZSPROC,DISPLAY,CHECKS,STATE=EXCEPTION-LOW
F HZSPROC,DISPLAY,CHECKS,STATUS=DELETED           # deleted checks
F HZSPROC,DISPLAY,CHECKS,STATUS=INACTIVE          # inactive checks
F HZSPROC,DISPLAY,CHECKS,CHECK=(name,owner)       # specific check
F HZSPROC,DISPLAY,CHECKS,OWNER=IBMRSM             # by owner
F HZSPROC,DISPLAY,CHECKS,SEVERITY=HIGH
F HZSPROC,DISPLAY,PARMLIB                         # HZSPRMxx concatenation
F HZSPROC,DISPLAY,SETTINGS
F HZSPROC,DISPLAY,EXCEPTIONS,HISTORY              # recent history
```

### Mutation (out of scope — confirm with user)

- `F HZSPROC,ACTIVATE,CHECK=(name,owner)` — reactivate a deactivated check
- `F HZSPROC,DEACTIVATE,CHECK=(name,owner)` — turn off a check
- `F HZSPROC,REFRESH,CHECK=(name,owner)` — re-run a check now
- `F HZSPROC,UPDATE,CHECK=(name,owner),PARM=(...)` — override check parms
- `F HZSPROC,DELETE,CHECK=(name,owner)` — remove check
- `F HZSPROC,ADDNEW` — reload HZSPRMxx
- `F HZSPROC,STOP` — stop the address space

---

## Batch — HZSPRINT

Print current results to SYSPRINT:

```jcl
//HZSCHK   JOB (ACCT),'HEALTH CHECK',CLASS=A,MSGCLASS=X,
//         NOTIFY=&SYSUID,REGION=0M
//STEP1    EXEC PGM=HZSPRNT,REGION=0M
//HZSPDATA DD DISP=SHR,DSN=HZS.HZSPDATA
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  LIST EXCEPTIONS
/*
```

Alternate SYSIN verbs:
- `LIST EXCEPTIONS` — only exceptions (recommended for automation)
- `LIST ALL` — every check + result
- `LIST BY(SEVERITY=HIGH)` — filter by severity
- `LIST BY(STATE=EXCEPTION-HIGH)` — high-severity exceptions only
- `LIST BY(CATEGORY=cat)` — by category tag
- `LIST BY(CHECK=(name,owner))` — one check
- `LIST BY(OWNER=owner)` — by owner (IBMRSM, IBMGRS, IBMXCF, etc.)
- `LIST HISTORY` — historical run info
- `DELETE HISTORY EXCEPT AGE(30)` — housekeeping (MUTATION on HZSPDATA)

Output format per check:
```
Check(IBMRSM,RSM_HVSHARE_SUPPRESS)             LAST RUN: yyyy-mm-dd hh:mm:ss.n
Owner:    IBMRSM
Category: MVS,RSM
Severity: MEDIUM
State:    ACTIVE(ENABLED)
Result:   SUCCESSFUL
Reason:   The check ran and reported no exceptions.

Check(IBMOMVS,BPXMCS_LOCKED_FILES)             LAST RUN: ...
Result:   EXCEPTION-MED
Message:  BPXH009E One or more files are locked...
Explanation: ...
System Action: ...
Operator Response: ...
Problem Determination: ...
Source: BPXOSMFP module
Reference: z/OS UNIX System Services Planning
```

Every field is machine-parseable — 200-char SYSPRINT records with clear labels.

---

## Categories to watch

The 200+ checks are grouped by owner. High-value owners for a site health check:

| Owner | What it covers |
|---|---|
| `IBMXCF` | Sysplex, XCF, CFRM/SFM/ARM policies |
| `IBMWLM` | WLM policy sanity, service class alignment |
| `IBMRSM` | Real Storage Manager (page dataset, HVSHARE, large frames) |
| `IBMRACF` | RACF settings (SETROPTS, class active/inactive, KDFAES) |
| `IBMASM` | Aux Storage Manager (paging headroom) |
| `IBMSVA` | System Virtual Assembly |
| `IBMOMVS` | z/OS UNIX (BPXPRMxx, mount options, thread limits) |
| `IBMGRS` | GRS mode + config |
| `IBMSMS` | DFSMS storage groups, ACS routines |
| `IBMLOGR` | System Logger + log stream health |
| `IBMTCP` | TCP/IP stacks, ports, listeners |
| `IBMUSS` | zFS/HFS mount config |
| `IBMPDSE` | PDSE latch health |
| `IBMCONSOLE` | Console configuration, MSCOPE |
| `IBMSMS` | SMS management |
| `IBMJES2` / `IBMJES3` | JES health |
| `IBMHSM` | HSM (DFSMShsm) config |
| `IBMICSF` | ICSF cryptographic health |

---

## Common exceptions to know

| Check name | Exception meaning |
|---|---|
| `RSM_HVSHARE_SUPPRESS` | Shared memory objects too large |
| `ASM_LOCAL_SLOT_USAGE` | Local page slot > 30% used — increase page datasets |
| `ASM_PLPA_COMMON_SIZE` | PLPA/common page slot > 50% used |
| `IEA_ASID_REUSE` | ASID reuse count low |
| `CNZ_CONSOLE_MASTERAUTH` | Too many consoles with MASTER authority |
| `CNZ_AMRF_EVENTUAL_ACTION` | AMRF (Action Message Retention Facility) not enabled |
| `CNZ_OPERLOG_SETUP` | OPERLOG not configured/active |
| `CNZ_SYSLOG_ALLOCATION` | SYSLOG allocation misconfigured |
| `RRS_STORAGE` | RRS storage utilization high |
| `RRS_RM_DATA_LOG_AVAILABLE` | RRS RM log unavailable |
| `PDSE_SMSPDSE1` | SMSPDSE1 latch contention |
| `SMS_CDS_SEPARATE_VOLUMES` | SMS CDS on same volume as primary (SPOF) |
| `SDSF_SERVER_PROC` | SDSF server proc not running |
| `USS_AUTOMOUNT_DELAY` | Automount policy delay too long |
| `USS_FILESYS_CONFIG_FILES` | Mount config file issues |
| `USS_HFS_DETECTED` | HFS still in use (should migrate to zFS) |
| `XCF_CDS_SPOF` | XCF CDS single point of failure |
| `XCF_CF_STR_AVAILABILITY` | Structure not defined as REBUILD-able |
| `XCF_CF_STR_PREFLIST` | Structure PREFLIST has 1 CF (SPOF) |
| `XCF_CF_MEMORY_UTILIZATION` | CF memory usage high |
| `XCF_CLEANUP_VALUE` | CLEANUP interval too high |
| `XCF_SIG_STR_SIZE` | XCF signaling structure sized wrong |
| `XCF_SFM_ACTIVE` | SFM policy not active |
| `WLM_STANDARD_CLASSES` | Missing standard WLM classes |
| `WLM_SYSTEM_RESOURCE_MANAGER` | SRM parms mis-set |
| `IEA_LXRES_NUMBER` | LX (linkage index) reserve low |

---

## Automated interpretation pattern

For the SHC coletor `phase_healthchk`:

1. Submit `HZSPRNT LIST EXCEPTIONS` JCL.
2. Copy SYSPRINT to USS via `OUTPUT` command or SDSF XDC.
3. Parse into CSV: `check_name, owner, category, severity, state, message`.
4. In report v6.5-MF, emit findings:
   - `EXCEPTION-HIGH` → HIGH severity finding
   - `EXCEPTION-MED` → MEDIUM
   - `EXCEPTION-LOW` → LOW
   - Attach message + reference (Redbook link)

Fallback if `HZS.HZSPDATA` READ denied:
- Try `F HZSPROC,DISPLAY,CHECKS,STATE=EXCEPTION` via ULOG.
- If ULOG also denied → mark evidence UNAVAILABLE and provide RACF profile guidance.

---

## Anti-patterns

- **Ignoring MEDIUM/LOW exceptions** — many turn into HIGH later. Treat every non-SUCCESSFUL as a signal.
- **Deactivating a noisy check** rather than fixing the root cause — DEACTIVATE hides the finding, doesn't fix the risk. If a check is not applicable, use `F HZSPROC,DEACTIVATE,POLICY=name` under change control.
- **Only running HZSPRNT once** — many checks are interval-based. Some fire only on state change. Look at `LAST RUN` timestamps.
- **HZSPRINT without `LIST EXCEPTIONS`** — dumps 200+ checks. In CI/automation, always filter.
- **Assuming HZSPROC address space is running** — some sites do not auto-start it. `D A,HZSPROC` is your friend.
