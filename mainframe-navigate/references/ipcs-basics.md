# IPCS Basics — z/OS Dump Analysis

IPCS (Interactive Problem Control System) is the tool for analyzing SVC dumps, stand-alone dumps, and SYSMDUMPs. Full IPCS mastery is a career; this reference is the minimum viable subset for a SME to open a dump, find failing code, and produce a triage summary.

**Read-only by definition.** IPCS reads dump datasets and generates reports. It never modifies live storage.

Related: [`abend-triage.md`](abend-triage.md) for pre-dump analysis via JESMSGLG/CEEDUMP.

---

## Enter IPCS

```
mainframe_send fieldName=OPTION text="6" aid=ENTER
mainframe_send text="IPCS" aid=ENTER
```

Or from ISPF Primary: option `IP` (site-defined shortcut) or option `S.IPCS`.

Primary panel offers:
```
0  DEFAULTS       - Set defaults (dsname, format, source description)
1  BROWSE         - Browse dump storage
2  ANALYSIS       - Formatted reports
3  UTILITY        - Dump housekeeping
4  INVENTORY      - Manage dump inventory
5  SUBMIT         - Batch IPCS
6  COMMAND        - Enter IPCS subcommands directly
```

---

## Set the dump source (option 0)

Every IPCS session needs a **source description** — the DSN of the dump you're analyzing:

```
mainframe_send fieldName=OPTION text="0" aid=ENTER
# In DEFAULTS panel:
mainframe_send labelLeft="Source" text="DSNAME('SYS1.DUMP01')" aid=NONE
mainframe_send labelLeft="Address Space" text="ASID(X'0043')" aid=ENTER
```

Persists for the session. Multi-dump comparison: define multiple sources and switch.

For an SVC dump captured to a specific dataset:
```
Source  ==> DSNAME('SYS1.DUMP01')
```

For a live system:
```
Source  ==> ACTIVE
```

---

## Option 6 — the IPCS command line

Most SME work happens in COMMAND mode. Key commands (case-insensitive):

### Status & summary

```
IP STATUS FAILDATA                     # what failed (PSW, ABEND code, module)
IP STATUS REGISTERS                    # GPR/FPR/CR at ABEND
IP STATUS WORKSHEET                    # comprehensive summary
IP STATUS SYSTEM                       # SYSPLEX, IPL time, system name
IP STATUS CPU                          # per-CP state
IP SUMMARY FORMAT                      # module/dsn map of the failing address space
IP SUMMARY REGS                        # register save area chain
IP SUMMARY TCBERROR                    # TCB chain with failing TCB flagged
```

`IP STATUS FAILDATA` is your **first command** — it prints:
- PSW (Program Status Word) — instruction pointer at ABEND
- ABEND code and reason
- Failing module name + offset
- Failing CSECT

### Where's the failing instruction?

```
IP WHERE X'8B4C0000'                   # what module/CSECT lives at this address?
IP WHERE PSWADDR                       # same, using PSW variable
IP LIST X'8B4C0000' LENGTH(64) INSTRUCTION      # disassemble around
```

### Walk TCB / RB chain

```
IP SUMMARY TCBERROR                    # find failing TCB
IP TCBSUMM ASID(X'0043')               # TCB list
IP RBSUMM TCB(X'009D2E48')             # RB chain for a TCB
IP VERBX MTRACE                        # system trace table (recent events)
```

### Dump storage examination

```
IP LIST X'0A000000' LENGTH(256) CHARACTER    # hex + char dump
IP LIST TCB(X'009D2E48') STRUCTURE(TCB)      # formatted TCB
IP LIST RB(X'009D3018') STRUCTURE(RB)        # formatted RB
IP LIST CVT STRUCTURE(CVT)                   # Communications Vector Table (system anchor)
IP CBFORMAT TCB(X'009D2E48') STRUCTURE(TCB)  # formatted control block
```

`STRUCTURE(name)` uses the CBFORMAT model definitions — z/OS ships with hundreds. Common:
- `TCB` — Task Control Block
- `RB` — Request Block
- `PRB` / `IRB` / `SVRB` — RB subtypes
- `ASCB` — Address Space Control Block
- `ASSB` — Auxiliary Storage Segment Block
- `CVT` — Communications Vector Table
- `PSA` — Prefix Save Area (CPU-local, low storage)
- `LDA` / `RBA` — Local Data Area / Region control
- `OUCB` — WLM Output Unit Control Block (per address space)

### LE / Language Environment dump

```
IP VERBX LEDATA                        # LE-specific report (heap, stack, condition)
IP VERBX LEDATA 'ALL'                  # comprehensive
IP VERBX LEDATA 'TRACEBACK ALL'        # every enclave's traceback
IP VERBX LEDATA 'HEAP'                 # heap analysis
IP VERBX LEDATA 'CEEDUMP'              # equivalent of CEEDUMP DD
```

### CICS / DB2 / IMS specific

```
IP VERBX CICSDATA                      # CICS system dump
IP VERBX DSNWDMP                       # DB2 dump (needs DSNWDMP module)
IP VERBX IMSDUMP                       # IMS
```

### System trace and GTF

```
IP VERBX MTRACE                        # system trace table
IP VERBX GTFTRACE                      # GTF trace records (if captured)
IP SLIPTRACE                           # SLIP-generated data
```

### RSM / storage manager

```
IP RSMDATA SUMMARY                     # real storage state
IP RSMDATA HIGH                        # high-water usage
IP FRAMES ASID(X'0043')                # frame allocation for ASID
```

### Enqueue / lock

```
IP ENQ                                 # global ENQ table at dump time
IP ENQ WAITERS                         # only enqueues with waiters
IP GRSDATA                             # GRS state
```

### Output redirection

```
IP SET PRINT ON                        # send output to IPCSPRNT
IP SET PRINT OFF
IP CLOSE PRINT                         # close IPCSPRNT dataset
```

For batch IPCS reports, define IPCSPRNT DD in JCL.

---

## Typical triage workflow

Given an SVC dump captured for ABEND S0C4:

```
1.  IP STATUS FAILDATA                 # PSW, ABEND, module
2.  IP STATUS REGISTERS                # GPR at fail
3.  IP SUMMARY FORMAT                  # module map (find your program)
4.  IP WHERE PSWADDR                   # what did we blow up in?
5.  IP LIST PSWADDR LENGTH(32) INSTRUCTION      # disassemble around failure
6.  IP SUMMARY TCBERROR                # failing TCB
7.  IP RBSUMM TCB(<TCB>)               # RB chain (who called who)
8.  IP VERBX LEDATA 'TRACEBACK ALL'    # LE traceback if COBOL/C/C++
9.  IP VERBX MTRACE                    # last N events before ABEND
```

Add DB2/CICS specific commands as needed.

---

## Batch IPCS

```jcl
//IPCS     EXEC PGM=IKJEFT01,DYNAMNBR=20,REGION=0M
//STEPLIB  DD DISP=SHR,DSN=SYS1.MIGLIB
//SYSTSPRT DD SYSOUT=*
//SYSTSIN  DD *
 IPCS NOPARM
 SETDEF DSNAME('SYS1.DUMP01') NOCONFIRM
 STATUS FAILDATA
 STATUS REGISTERS
 SUMMARY FORMAT
 SUMMARY TCBERROR
 VERBX LEDATA 'TRACEBACK ALL'
 END
/*
//IPCSPRNT DD SYSOUT=*
//IPCSDDIR DD DISP=SHR,DSN=IPCS.DIR
//IPCSPARM DD DISP=SHR,DSN=IPCS.PARMLIB
```

`IPCS.DIR` is site-specific — check `LISTC LVL('IPCS')` or ask sysprog.

---

## Dump inventory

```
mainframe_send fieldName=OPTION text="4" aid=ENTER    # from IPCS primary
```

Lists SYS1.DUMPnn datasets on this LPAR and their capture state (in-use / captured / free). MUTATION verbs on this panel (`D`, `E`) are out of scope — never delete a dump without SME approval; dumps are evidence.

Alternate: `D DUMP,STATUS` and `D DUMP,TITLE=ALL` from ULOG.

---

## Modeling SVC dump capture

When you need a dump of a running problem (SME task), the operator can issue `DUMP TITLE=(...),JOBNAME=(...),SDATA=(...)`. This IS a mutation (allocates DASD, spends system resources) — **out of scope for this skill**, but understanding IPCS ingests them is useful.

---

## Anti-patterns

- **`IP LIST X'0'` on live system** — dumps low storage which contains PSA. Fine, but voluminous. Better: `IP LIST PSA STRUCTURE(PSA)`.
- **Random `VERBX` without knowing what's in the dump** — `VERBX LEDATA` on a JES2 dump gets you nothing. Match VERBX to dump kind.
- **Skipping `IP STATUS FAILDATA`** — always the first command. Everything else is context.
- **Working in BROWSE mode for triage** — BROWSE is for byte-level inspection. Use COMMAND mode for formatted reports.
- **Deleting dumps without archiving** — dumps ARE the evidence. Site policy usually mandates 30/60/90-day retention. Confirm before D on option 4.
