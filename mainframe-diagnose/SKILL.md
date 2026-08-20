---
name: mainframe-diagnose
description: z/OS mainframe and Unix System Services specialist. Diagnoses system issues including JCL failures, abends, USS filesystem problems, batch job errors, DASD space, DB2 connectivity, CICS regions, catalog issues, GDG management, and performance bottlenecks. Provides expertise in JCL, REXX, TSO commands, ISPF utilities, SDSF, USS shell, and z/OS system internals.
version: 1.0.0
author: ACI PS CMM
tags: [mainframe, z/os, jcl, rexx, tso, ispf, sdsf, uss, omvs, db2, cics, batch, abend]
category: diagnose
---

# /mainframe-diagnose — z/OS Mainframe & USS Specialist

## When to use

- JCL job failures (JCL errors, abends, condition codes)
- Batch processing issues (step failures, dataset allocation, GDG overflow)
- USS/OMVS problems (filesystem full, permissions, encoding)
- DASD space issues (extent failures, SMS class problems)
- Catalog errors (VSAM, ICF catalog issues)
- DB2 connectivity from batch (DSNTIAG, BIND failures, plan not found)
- CICS region issues (transaction dumps, short-on-storage, maxqueue)
- Performance (CPU spin, wait states, enqueue contention)
- REXX/TSO scripting questions

---

## Step 1 — Identify the Problem Class

Ask the user which area is failing:

| Problem | Key Indicators | Jump to |
|---------|---------------|---------|
| JCL error | JCL ERROR message, job not starting | Step 2 |
| Abend in step | Sxxx or Uxxx code | Step 3 |
| Non-zero condition code | COND CODE != 0000 | Step 4 |
| Dataset allocation failure | IEF257I, IGD17xxxI | Step 5 |
| USS/OMVS issue | EDC/FSUM messages, permission denied | Step 6 |
| DB2 failure | DSNT messages, -805/-811/-904 sqlcode | Step 7 |
| CICS issue | DFHAC/DFHSM messages, transaction dump | Step 8 |
| Performance | Job running too long, CPU time excessive | Step 9 |

---

## Step 2 — JCL Errors

Common JCL error messages and fixes:

| Message | Cause | Fix |
|---------|-------|-----|
| IEFC001I | Invalid JCL statement | Check syntax — missing comma, invalid keyword |
| IEFC621I | Unmatched quote/paren | Find unclosed string in JCL |
| IEFC630I | Invalid keyword | Typo in DD parameter |
| IEFC452I | Invalid operand | Check SPACE, DCB, or DISP parameters |
| IEF212I | Dataset not found | Verify DSN exists in catalog |
| IEF257I | Dataset not available | Check DISP — someone else has exclusive |

### JCL validation checklist
```
1. Every JOB card has CLASS, MSGCLASS, MSGLEVEL
2. Every EXEC has PGM= or PROC=
3. Every DD has DSN= and DISP=
4. LRECL/RECFM/BLKSIZE match between input and output
5. Concatenations share same RECFM (or use LIKE=)
6. COND/IF-THEN-ELSE logic is correct
7. Symbolic parameters resolved (&VAR syntax)
```

---

## Step 3 — System and User Abends

### System abends (Sxxx)

| Abend | Meaning | Action |
|-------|---------|--------|
| S001 | I/O error | Check dataset — run IEHLIST, verify volume |
| S013 | Dataset conflict (LRECL/BLKSIZE mismatch) | Verify DCB parameters match actual dataset |
| S0C1 | Operation exception | Program trying to execute invalid instruction — check load module |
| S0C4 | Protection exception | Program accessing storage outside its region — check pointer/subscript |
| S0C7 | Data exception | Packed decimal field contains non-numeric — check input data |
| S0CB | Division by zero | Check denominator before DIVIDE |
| S106 | Module not found | Verify STEPLIB/JOBLIB has the load library |
| S213 | Dataset not found on volume | Catalog points to wrong volume |
| S222 | Job cancelled by operator | Check with operations — was it intentional? |
| S322 | Time exceeded (CPU or wait) | Increase TIME on JOB/EXEC or fix loop |
| S806 | Module not in load library | Check STEPLIB concatenation |
| S878/80A | Insufficient virtual storage | Increase REGION on JOB card |
| SB37/D37/E37 | Dataset out of space | Add secondary allocation or allocate larger primary |

### User abends (Uxxx)

| Abend | Typical source | Action |
|-------|---------------|--------|
| U0016 | SORT — invalid control statement | Check SORT SYSIN |
| U1026 | IDCAMS — VSAM error | Check IDCAMS SYSPRINT for return code |
| U4038 | COBOL — file status error | Check FILE STATUS variable — usually 35 or 39 |
| U4093 | Language Environment — unhandled condition | Check LE dump (CEEDUMP) |

---

## Step 4 — Condition Code Analysis

| Return Code | Meaning |
|-------------|---------|
| 0 | Success |
| 4 | Warning — review output but usually OK |
| 8 | Error — something failed, check SYSPRINT |
| 12 | Severe error — job logic likely wrong |
| 16 | Fatal — processing cannot continue |
| >16 | Program-specific — check program documentation |

### Checking condition codes in JCL
```jcl
//STEP02   EXEC PGM=MYPROG,COND=(4,LT)     ← skip if prior RC < 4
//STEP03   EXEC PGM=NEXT,COND=(0,NE,STEP01) ← skip unless STEP01 RC = 0
//*
//* IF/THEN/ELSE (modern JCL):
// IF (STEP01.RC = 0) THEN
//STEP02   EXEC PGM=GOOD
// ELSE
//STEP02B  EXEC PGM=RECOVER
// ENDIF
```

---

## Step 5 — Dataset Allocation Failures

| Message | Cause | Fix |
|---------|-------|-----|
| IEF257I | Dataset in use exclusively | Wait or check who has it (SDSF DA, D GRS) |
| IGD17001I | SMS rejected — no suitable volume | Check SMS class, ACS rules, or use specific VOLUME |
| IGD17101I | Insufficient space on volume | Try different volume or increase allocation |
| IEC030I | OPEN error — ABEND S013 | LRECL/BLKSIZE mismatch |
| IEC141I | End of volume, no secondary space | Add SPACE secondary allocation |
| IEC143I | Insufficient space (SB37) | Compress PDS, or reallocate with more space |

### Space calculations
```
Tracks per cylinder: 15
Bytes per track (3390): 56,664
1 MB ≈ 18 tracks ≈ 1.2 cylinders

Common SPACE values:
  SPACE=(CYL,(10,5),RLSE)       — 10 primary, 5 secondary cylinders
  SPACE=(TRK,(100,50),RLSE)     — 100 primary, 50 secondary tracks
  SPACE=(27920,(1000,500),RLSE) — block-level allocation
```

---

## Step 6 — USS/OMVS Problems

### Common USS error codes

| Error | Message | Fix |
|-------|---------|-----|
| EDC5111I | Permission denied | Check file permissions (`ls -la`), RACF UNIXPRIV |
| EDC5112I | File not found | Verify path — USS is case-sensitive |
| EDC5129I | Disk space full | Check filesystem: `df -k /path` |
| FSUM7351 | Command not found | Not in PATH — use full path or update $PATH |
| FSUM6785 | File or directory not found | Typo in path or file doesn't exist |
| FSUM7332 | Permission denied (execute) | `chmod +x script.sh` |

### USS filesystem commands
```bash
# Check filesystem usage
df -k /path/to/filesystem

# Check user quota
quota

# Find large files
find /path -size +10M -ls

# Check encoding (important for z/OS)
chtag -p filename           # shows file tag
chtag -t -c ISO8859-1 file  # tag as ASCII
chtag -t -c IBM-1047 file   # tag as EBCDIC

# File transfer between USS and MVS
cp "//'HLQ.DATASET(MEMBER)'" ./localfile    # MVS → USS
cp ./localfile "//'HLQ.DATASET(MEMBER)'"    # USS → MVS
```

### z/OS USS specifics
```bash
# Environment
echo $PATH            # Should include /bin:/usr/bin
echo $_BPXK_AUTOCVT  # Auto-convert setting (ON/OFF/ALL)
echo $_CEE_RUNOPTS    # LE runtime options

# Mount points
df -P                 # POSIX format, shows mount points
/usr/sbin/mount       # Show all mounts

# Process management
ps -ef                # All processes
kill -9 <pid>         # Force kill

# RACF (security)
id                    # Show UID/GID
groups                # Show group membership
```

---

## Step 7 — DB2 Failures from Batch

| SQLCODE | Meaning | Fix |
|---------|---------|-----|
| -805 | Program not found in plan | BIND the package/plan |
| -811 | Multiple rows returned for singleton SELECT | Add WHERE clause or use cursor |
| -818 | Timestamp mismatch | REBIND plan/package |
| -904 | Unavailable resource | DB2 tablespace in STOP or COPY pending |
| -911 | Deadlock/timeout | Check locking — review SQL access path |
| -922 | Authorization failure | GRANT required on table/plan |
| -923 | DB2 connection not available | Check DB2 subsystem status |
| -927 | Language interface not initialized | DSNALI missing from STEPLIB |

### DB2 batch JCL pattern
```jcl
//STEP01   EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD  DSN=DB2.SDSNLOAD,DISP=SHR
//         DD  DSN=your.loadlib,DISP=SHR
//SYSTSPRT DD  SYSOUT=*
//SYSPRINT DD  SYSOUT=*
//SYSTSIN  DD  *
  DSN SYSTEM(DB2P)
  RUN PROGRAM(MYPROG) PLAN(MYPLAN) -
      LIB('your.loadlib')
  END
/*
```

---

## Step 8 — CICS Issues

| Message prefix | Area | Check |
|----------------|------|-------|
| DFHAC | Access control | Security definitions, RACF |
| DFHAM | Application manager | Program/transaction definitions |
| DFHAP | Application | Program abends |
| DFHFC | File control | VSAM file issues |
| DFHSM | Storage manager | Short-on-storage, CUSHION |
| DFHTC | Terminal control | Network issues |
| DFHTR | Trace | Trace table entries |

### CICS commands (via TSO CEMT or the 3270 bridge)
```
CEMT I TRANS(xxxx)         — Inquire transaction status
CEMT I PROG(xxxxxxxx)     — Inquire program status
CEMT I FILE(xxxxxxxx)     — Inquire file status
CEMT S TRANS(xxxx) ENA     — Enable transaction
CEMT S PROG(xxxxxxxx) NEW  — New copy of program
CEMT I TASK               — Show active tasks
CEMT I SYS                — System status
```

---

## Step 9 — Performance Analysis

### Key indicators
```
CPU time vs. elapsed time ratio:
  > 0.9  = CPU bound (optimize code/SQL)
  < 0.1  = I/O or wait bound (check DASD, enqueue, DB2 locks)

SMF records for job analysis:
  SMF 30  — Job/step resource usage
  SMF 101 — DB2 accounting
  SMF 110 — CICS performance
```

### Common bottlenecks

| Symptom | Likely cause | Action |
|---------|-------------|--------|
| High CPU, low elapsed | CPU-bound program | Optimize loops, SQL, SORT |
| Low CPU, high elapsed | Waiting — I/O, locks, or ENQ | Check enqueue contention (D GRS) |
| Excessive EXCP count | Too many I/Os | Increase BLKSIZE, use BUFNO |
| DB2 thread wait | Lock contention | Check DB2 -DISPLAY THREAD |
| SORT taking too long | Large file, insufficient SORTWK | Add more SORTWK DDs, use DYNALLOC |

---

## REXX Quick Reference

```rexx
/* REXX - Template for TSO REXX exec */
/* Trace ?R for interactive debug    */
SAY "Starting..."

/* Read a dataset */
"ALLOC F(INFILE) DA('HLQ.DATASET') SHR REUSE"
"EXECIO * DISKR INFILE (STEM line. FINIS"
"FREE F(INFILE)"

DO i = 1 TO line.0
  SAY line.i
END

/* Execute TSO commands */
ADDRESS TSO
"LISTDS 'HLQ.DATASET' MEMBERS"

/* ISPF services from REXX */
ADDRESS ISPEXEC
"LMINIT DATAID(did) DATASET('HLQ.PDS')"
"LMOPEN DATAID("did") OPTION(INPUT)"
"LMMLIST DATAID("did") OPTION(LIST) MEMBER(mem)"
DO WHILE RC = 0
  SAY mem
  "LMMLIST DATAID("did") OPTION(LIST) MEMBER(mem)"
END
"LMCLOSE DATAID("did")"
"LMFREE DATAID("did")"

EXIT 0
```

---

## TSO/ISPF Command Reference

| Command | Description |
|---------|-------------|
| `LISTDS 'dsn' MEMBERS` | List PDS members |
| `LISTDS 'dsn' STATUS` | Show dataset allocation status |
| `LISTCAT ENT('dsn') ALL` | Full catalog entry details |
| `DELETE 'dsn'` | Delete a dataset |
| `RENAME 'old' 'new'` | Rename dataset |
| `SUBMIT 'dsn(member)'` | Submit JCL |
| `STATUS jobname` | Check job status |
| `CANCEL jobname` | Cancel running job |
| `OUTPUT jobname(jobid)` | View job output |
| `PROFILE PREFIX(hlq)` | Set TSO prefix |
| `ALLOCATE ...` | Allocate new dataset |
| `FREE F(ddname)` | Free allocation |

---

## GDG (Generation Data Group) Management

```jcl
//* Define GDG base (IDCAMS)
//DEFGDG   EXEC PGM=IDCAMS
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  DEFINE GDG(NAME('HLQ.MYGDG') LIMIT(10) SCRATCH NOEMPTY)
/*

//* Reference current generation
//INPUT    DD DSN=HLQ.MYGDG(0),DISP=SHR

//* Create new generation
//OUTPUT   DD DSN=HLQ.MYGDG(+1),DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(10,5),RLSE),
//            DCB=(RECFM=FB,LRECL=80,BLKSIZE=27920)

//* Previous generation
//PREV     DD DSN=HLQ.MYGDG(-1),DISP=SHR
```

---

## Useful SDSF Commands

| Command | What it does |
|---------|-------------|
| `ST` | Active jobs |
| `H` | Held output queue |
| `O` | Output queue |
| `DA` | Display active address spaces |
| `LOG` | System log |
| `SYSLOG` | System log (full) |
| `PREFIX jobname` | Filter by job prefix |
| `OWNER userid` | Filter by owner |
| `S` (next to job) | View SYSOUT |
| `?` (next to job) | Job details |
| `P` (next to job) | Purge output |
| `C` (next to job) | Cancel job |
