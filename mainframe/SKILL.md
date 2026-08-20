---
name: mainframe
description: Senior specialist in IBM z/OS mainframe systems and Unix System Services (USS/OMVS). Compiled knowledge from IBM Redbooks ABCs of z/OS System Programming (13 volumes), z/OS MVS JCL Reference, z/OS UNIX System Services Planning/User's Guide, DFSMS manuals, RACF Security Admin Guide, WLM documentation, and hands-on production experience. Covers z/OS internals, JCL, TSO/ISPF, SDSF, DFSMS/SMS, VSAM, catalogs, JES2/JES3, TCP/IP, VTAM, Parallel Sysplex, RACF security, problem diagnosis (dumps/traces/IPCS), USS shell/filesystem/security, z/Architecture, LPAR/HCD, performance/WLM/RMF/SMF, and SMP/E maintenance. Use for any question about z/OS, USS, JCL, batch processing, mainframe storage, security, networking, performance, or system programming.
---

# IBM z/OS Mainframe & Unix System Services — Senior Specialist

> Compiled knowledge from IBM Redbooks "ABCs of z/OS System Programming" (13 volumes, SG24-6981 through SG24-6992 + SG24-6327), z/OS MVS JCL Reference (SA23-1385), z/OS UNIX System Services Planning (GA32-0884), DFSMS manuals, RACF Security Administrator's Guide (SA23-2289), and production experience on z/OS 2.4/2.5 LPARs.

---

## Response Guidelines

When explaining multi-component flows, storage hierarchies, Sysplex topology, or JES spool flows, include a **plain-text diagram inside a fenced code block** using box-drawing characters (`│ ├── └── → ▼`), or a Markdown table for relationships. **Do NOT use Mermaid** — the Claude Code chat does not render it (the source code is shown raw to the user). For simple command/parameter questions, text-only is sufficient.

Example shape for a JES spool flow:

```
JCL ──► [ JES2 reader ] ──► Initiator ──► Step (DD/DSN) ──► SYSOUT
              │
              └── DFSMS (SMS class / VSAM cluster)
```

---

## 1. z/OS Architecture Overview (Vol 1, 10)

### 1.1 z/Architecture Fundamentals

```
┌─────────────────────────────────────────────────────────┐
│                   Physical CPC (z16/z15/z14)            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  LPAR 1  │  │  LPAR 2  │  │  LPAR 3  │  ...        │
│  │  (z/OS)  │  │  (z/OS)  │  │  (Linux)  │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│         PR/SM (Processor Resource/Systems Manager)      │
│         HMC (Hardware Management Console)               │
│         SE (Support Element)                            │
└─────────────────────────────────────────────────────────┘
```

| Concept | Description |
|---------|-------------|
| CPC | Central Processor Complex — the physical machine |
| LPAR | Logical Partition — virtualized z/OS instance (PR/SM) |
| CP | Central Processor — general purpose engine |
| zIIP | z Integrated Information Processor — offload eligible work (DB2, XML, IPSec) |
| zAAP | z Application Assist Processor — Java workloads (deprecated → zIIP) |
| ICF | Internal Coupling Facility — Sysplex coupling |
| SAP | System Assist Processor — I/O subsystem |

### 1.2 Addressing Modes

| Mode | Bits | Max addressable | PSW bit |
|------|------|-----------------|---------|
| 24-bit | 24 | 16 MB ("below the line") | AMODE 24 |
| 31-bit | 31 | 2 GB ("above the line") | AMODE 31 |
| 64-bit | 64 | 16 EB ("above the bar") | AMODE 64 |

### 1.3 Storage Hierarchy

```
┌─────────────────────────────────────────┐
│ Real Storage (physical memory)          │
├─────────────────────────────────────────┤
│ 0–16MB: Common below-the-line (24-bit)  │
│ 16MB–2GB: Extended private/common       │
│ 2GB+: 64-bit common/private ("bar")     │
└─────────────────────────────────────────┘

Address Space Layout:
┌────────────────┐ 2GB
│ Extended LSQA  │
│ Extended Priv  │
│ Extended CSA   │
├────────────────┤ 16MB ("the line")
│ LSQA           │
│ Private area   │
│ CSA            │
│ SQA            │
│ Nucleus        │
│ PSA (page 0)   │
└────────────────┘ 0
```

| Area | Full name | Purpose |
|------|-----------|---------|
| PSA | Prefixed Save Area | Per-CPU; interrupt vectors, current TCB/ASCB pointers |
| Nucleus | z/OS Nucleus | Kernel; loaded at IPL from SYS1.NUCLEUS |
| SQA | System Queue Area | System-wide control blocks (no page-out) |
| CSA | Common Service Area | Shared across address spaces (pageable) |
| LSQA | Local System Queue Area | Per-address-space control blocks |
| Private | Private area | User region — programs, data, getmain |
| LPA | Link Pack Area | Shared read-only modules (SVC routines, ISPF) |
| PLPA | Pageable LPA | Pageable shared modules |
| FLPA | Fixed LPA | Non-pageable shared modules |
| MLPA | Modified LPA | Temporary overrides (testing) |

### 1.4 IPL Process

```
HMC → Load → IML (Initial Microcode Load)
  → LOD (Load from IODF — device configuration)
  → NIP (Nucleus Initialization Program)
    → reads LOADxx parmlib member
    → reads IEASYSxx (system parameters)
    → reads IEASVCxx (SVC definitions)
    → initializes RSM, ASM, SRM, WLM, JES
  → Master Scheduler
    → starts system address spaces (CONSOLE, GRS, VTAM, TCP/IP, JES2/3)
    → reads COMMNDxx (auto-start commands)
    → processes automation (SA z/OS or INITTAB)
```

Key parmlib members for IPL:

| Member | Controls |
|--------|----------|
| LOADxx | IODF suffix, NUCLEUS dsn, IEASYM symbols |
| IEASYSxx | Master system parameters (CSA, SQA, REGION, MAXUSER) |
| IEASVCxx | SVC table entries |
| PROGxx | APF list, LPA list, LNKLST |
| COMMNDxx | Auto-start commands at IPL |
| IKJTSOxx | TSO parameters |
| SMFPRMxx | SMF recording parameters |
| IEAFIXxx | Fixed pages |
| COUPLExx | Sysplex Coupling Facility definitions |

---

## 2. TSO/E, ISPF, JCL, SDSF (Vol 1)

### 2.1 TSO/E (Time Sharing Option/Extended)

TSO is the interactive command-line interface to z/OS. It is the base layer under ISPF.

Key TSO commands:

| Command | Purpose |
|---------|---------|
| `LOGON` / `LOGOFF` | Start/end TSO session |
| `LISTDS 'dsn' MEMBERS` | List PDS members |
| `LISTDS 'dsn' STATUS` | Show allocation status |
| `LISTCAT ENT('dsn') ALL` | Catalog details (VSAM clusters too) |
| `DELETE 'dsn'` | Delete dataset |
| `RENAME 'old' 'new'` | Rename |
| `ALLOCATE` | Allocate (create) new dataset |
| `FREE F(dd)` | Release DD allocation |
| `SUBMIT 'dsn(mbr)'` | Submit JCL job |
| `STATUS jobname` | Check job status |
| `CANCEL jobname` | Cancel running job |
| `EXEC 'dsn(rexx)' 'args'` | Run REXX exec |
| `PROFILE PREFIX(hlq)` | Set default HLQ |
| `SEND 'msg' USER(userid)` | Send message to another user |

### 2.2 ISPF Panel Navigation

| Option | Function |
|--------|----------|
| 0 | Settings |
| 1 | View (read-only browse) |
| 2 | Edit |
| 3 | Utilities |
| 3.1 | Library utility (PDS) |
| 3.2 | Dataset utility |
| 3.3 | Move/Copy |
| 3.4 | Dataset list (DSLIST) — most used |
| 4 | Foreground compile |
| 5 | Batch (submit) |
| 6 | TSO Command Shell |
| S | SDSF |
| =X | Exit ISPF completely |

ISPF Edit primary commands:

| Command | Purpose |
|---------|---------|
| `SAVE` | Save without exit |
| `END` | Save and exit |
| `CANCEL` | Exit without saving |
| `FIND 'text'` | Search forward |
| `CHANGE 'old' 'new' ALL` | Replace all |
| `SUBMIT` | Submit as JCL |
| `COPY AFTER .x` | Copy after label |
| `SORT` | Sort lines |
| `RESET` | Reset line commands |
| `HILITE COBOL` / `JCL` / `REXX` | Syntax highlighting |

ISPF Edit line commands: `I` (insert), `D` (delete), `R` (repeat), `C`/`M` (copy/move), `A`/`B` (after/before), `COLS` (show columns), `BNDS` (show boundaries).

### 2.3 JCL — Job Control Language

```jcl
//JOBNAME  JOB (acct),'pgmr name',
//         CLASS=A,MSGCLASS=X,MSGLEVEL=(1,1),
//         REGION=0M,TIME=1440,NOTIFY=&SYSUID
//*-------------------------------------------------------------------
//STEP01   EXEC PGM=IEFBR14
//NEWDS    DD  DSN=HLQ.NEW.DATASET,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(10,5),RLSE),
//            DCB=(RECFM=FB,LRECL=80,BLKSIZE=27920)
//*-------------------------------------------------------------------
//STEP02   EXEC PGM=IEBCOPY
//SYSPRINT DD  SYSOUT=*
//SYSUT1   DD  DSN=HLQ.SOURCE.PDS,DISP=SHR
//SYSUT2   DD  DSN=HLQ.TARGET.PDS,DISP=SHR
//SYSIN    DD  *
  COPY OUTDD=SYSUT2,INDD=SYSUT1
  SELECT MEMBER=MYMEMBER
/*
```

#### JCL Parameters Reference

| JOB Parameter | Values | Purpose |
|---------------|--------|---------|
| CLASS | A-Z, 0-9 | Job class (execution priority/category) |
| MSGCLASS | A-Z, 0-9 | Output class for JES messages |
| MSGLEVEL | (stmt,msg) | (1,1)=all JCL+messages; (0,0)=none |
| REGION | nM / nK / 0M | Private area size; 0M=unlimited |
| TIME | (min,sec) / 1440 / NOLIMIT | CPU time limit; 1440=no limit |
| NOTIFY | &SYSUID | Send completion message to userid |
| TYPRUN | SCAN / HOLD / COPY | Scan JCL only / hold / copy SYSOUT |
| RESTART | stepname | Restart from specific step |
| COND | (code,op) | Skip step conditionally |

| DD Parameter | Values | Purpose |
|--------------|--------|---------|
| DSN | dataset.name | Dataset name (max 44 chars) |
| DISP | (status,normal,abnormal) | NEW/OLD/SHR/MOD; CATLG/KEEP/DELETE/PASS |
| SPACE | (unit,(pri,sec,dir)) | CYL/TRK/blksize; primary, secondary, directory blocks |
| DCB | (RECFM,LRECL,BLKSIZE) | Data Control Block attributes |
| UNIT | SYSDA / 3390 / specific | Device type |
| VOL | SER=volser | Specific volume serial |
| SYSOUT | class | Direct output to JES spool |
| DUMMY | — | Null file (discard output / empty input) |
| * | — | Instream data follows |
| LIKE | dsn | Copy attributes from model |
| REFDD | *.stepname.ddname | Reference another DD's attributes |

#### DISP combinations (most common)

| DISP | Meaning |
|------|---------|
| `(NEW,CATLG,DELETE)` | Create new; catalog if OK, delete if abend |
| `(OLD,KEEP)` | Exclusive access; keep when done |
| `(SHR,KEEP)` | Shared read; keep |
| `(MOD,CATLG)` | Append (or create if not exists); catalog |
| `(NEW,PASS)` | Create temp; pass to next step |
| `(OLD,DELETE)` | Exclusive; delete when done |

#### Conditional Processing

```jcl
//* Classic COND parameter (skip if condition TRUE)
//STEP02   EXEC PGM=NEXT,COND=(4,LT)        ← skip if any prior RC < 4
//STEP03   EXEC PGM=NEXT,COND=(0,NE,STEP01)  ← skip unless STEP01 RC = 0

//* Modern IF/THEN/ELSE/ENDIF
// IF (STEP01.RC <= 4) THEN
//STEP02   EXEC PGM=GOODPATH
// ELSE
//STEP02E  EXEC PGM=ERRORPATH
// ENDIF
```

#### Common Utilities

| Utility | Purpose |
|---------|---------|
| IEFBR14 | Do-nothing (allocate/delete datasets via DD) |
| IEBCOPY | PDS copy/compress/unload |
| IEBGENER | Sequential copy/convert |
| IEBUPDTE | Create/update PDS members from card images |
| IDCAMS | VSAM utility (DEFINE, DELETE, REPRO, LISTCAT, ALTER) |
| SORT (DFSORT/ICETOOL) | Sort, merge, join, select, reformat |
| IKJEFT01 | TSO-in-batch (for REXX, DSN, ISPF) |
| IEHLIST / IEHPROGM | VTOC listing, scratch/rename datasets |
| ADRDSSU (DFSMSdss) | DUMP/RESTORE/COPY/DEFRAG physical datasets |
| AMBLIST | List load module contents |
| GIMSMP | SMP/E processing |

### 2.4 SDSF

| Command | What it does |
|---------|-------------|
| `ST` | Status of active jobs |
| `H` | Held output queue |
| `O` | Output queue |
| `DA` | Display active address spaces |
| `LOG` | System log (SYSLOG) |
| `PREFIX jobname*` | Filter jobs by prefix |
| `OWNER userid` | Filter by job owner |
| `S` (line cmd) | Browse SYSOUT/output |
| `?` (line cmd) | Job details (steps, RC) |
| `P` (line cmd) | Purge output |
| `C` (line cmd) | Cancel running job |
| `SJ` (line cmd) | Submit JCL (rerun) |

---

## 3. System Maintenance (Vol 2)

### 3.1 Parmlib and IPL

The system parameter library (`SYS1.PARMLIB`) controls ALL z/OS behavior. Multiple parmlib concatenations supported.

Key daily maintenance:

| Task | Command/Method |
|------|----------------|
| Display active parmlib | `D PARMLIB` |
| Change SMF options | `SET SMF=xx` |
| Refresh LLA (module cache) | `F LLA,REFRESH` |
| Refresh LPA | Requires IPL (or MLPA for testing) |
| Display system status | `D A,L` (active address spaces) |
| Display storage | `D VSM,ALLOC` |
| Cancel address space | `C jobname` / `FORCE jobname,ARM` |
| Start started task | `S procname` |
| Stop started task | `P procname` |

### 3.2 JES2 (Job Entry Subsystem)

JES2 manages job input, scheduling, and output.

```
Job Lifecycle:
  SUBMIT → INPUT queue → EXECUTION → OUTPUT queue → PURGE
                │                         │
                └── Conversion            └── Print/Spool
                    (JCL parsing)              (writers)
```

| JES2 Command | Purpose |
|--------------|---------|
| `$D JOBQ,JOBS=*` | Display job queue summary |
| `$D J'jobname'` | Display specific job |
| `$A J'jobname'` | Release held job |
| `$C J'jobname'` | Cancel job |
| `$P J'jobname'` | Purge job output |
| `$D INITQ` | Display initiator queues |
| `$S INIT(nn)` | Start initiator |
| `$T JOBn,C=class` | Change job class |
| `$D SPOOL` | Spool utilization |
| `$D OUTQ` | Output queue status |

### 3.3 LPA, LNKLST, and Authorized Libraries

| Library type | Description | How to update |
|-------------|-------------|---------------|
| LNKLST | System search path for load modules | PROGxx + `SET PROG=xx` or `SETPROG LNKLST,ADD` |
| LPA (PLPA+FLPA) | Shared reentrant modules (one copy for all) | Requires IPL (or MLPA for temp) |
| MLPA | Modified LPA — testing overrides | `SET PROG=xx` with MLPA entries |
| APF | Authorized Program Facility — programs can use authorized SVCs | PROGxx + `SETPROG APF,ADD` |
| STEPLIB/JOBLIB | Job-specific load libraries | DD in JCL |

### 3.4 SMP/E (System Modification Program/Extended)

SMP/E manages ALL software maintenance (PTFs, APARs, USERMODs).

```
               RECEIVE → APPLY → ACCEPT
                  │         │        │
                  ▼         ▼        ▼
               GLOBAL    TARGET   DLIB zones
              (catalog)  (test)   (permanent)
```

| SMP/E Command | Purpose |
|---------------|---------|
| `RECEIVE` | Load SYSMOD into global zone |
| `APPLY CHECK` | Dry-run — show what would change |
| `APPLY` | Install into target libraries |
| `ACCEPT CHECK` | Dry-run accept |
| `ACCEPT` | Make permanent (DLIB) |
| `RESTORE` | Back out an applied SYSMOD |
| `LIST SYSMODS` | Show installed maintenance |
| `REPORT CROSSZONE` | Show what's applied but not accepted |

---

## 4. DFSMS — Data Management (Vol 3)

### 4.1 Dataset Organization Types

| Type | Description | Access | Examples |
|------|-------------|--------|----------|
| PS (Sequential) | Records in order | Sequential only | Flat files, logs |
| PO (Partitioned — PDS) | Directory + members | Direct by member | Source libraries, JCL |
| PDSE (PDS/E) | Extended PDS — no compress needed | Direct by member | Preferred over PDS |
| VSAM KSDS | Key-Sequenced Data Set | Key, sequential, skip-sequential | DB files, tables |
| VSAM ESDS | Entry-Sequenced Data Set | Sequential, RBA access | Logs, journals |
| VSAM RRDS | Relative Record Data Set | Record number | Fixed-slot access |
| VSAM LDS | Linear Data Set | CI-level access | DB2 tablespaces, hiperspaces |
| DA (Direct) | Relative track/block | By address | Legacy — rare |
| HFS/zFS | Hierarchical/z File System | POSIX path | USS filesystems |

### 4.2 DCB Attributes

| Parameter | Values | Meaning |
|-----------|--------|---------|
| RECFM | F/FB/V/VB/FBA/VBA/U | Fixed/Variable/Undefined; B=blocked, A=ASA CC |
| LRECL | 1–32760 | Logical record length (V format: max incl 4-byte RDW) |
| BLKSIZE | ≤32760 (or 0=SDB) | Physical block size; 0=system-determined |
| DSORG | PS/PO/DA/VS | Dataset organization |
| KEYLEN | 1–255 | VSAM/ISAM key length |

### 4.3 SMS (System-Managed Storage)

SMS automates storage allocation through ACS (Automatic Class Selection) routines:

```
User requests SPACE → ACS routines fire:
  → STORCLAS (Storage Class) — performance/availability
  → MGMTCLAS (Management Class) — migration/backup/retention
  → DATACLAS (Data Class) — DCB/SPACE defaults
  → SGROUPNM (Storage Group) — which volumes
```

### 4.4 VSAM — IDCAMS Commands

```
//* Define a KSDS cluster
DEFINE CLUSTER(NAME('HLQ.VSAM.KSDS') -
       VOLUMES(VOL001) -
       RECORDS(10000 5000) -
       KEYS(8 0) -
       RECORDSIZE(100 200) -
       SHAREOPTIONS(2 3) -
       INDEXED) -
  DATA(NAME('HLQ.VSAM.KSDS.DATA') -
       CISZ(4096)) -
  INDEX(NAME('HLQ.VSAM.KSDS.INDEX') -
       CISZ(2048))

//* REPRO — copy data into/from VSAM
REPRO INFILE(INPUT) OUTFILE(OUTPUT)

//* LISTCAT — show catalog details
LISTCAT ENT('HLQ.VSAM.KSDS') ALL

//* ALTER — change attributes
ALTER 'HLQ.VSAM.KSDS' SHAREOPTIONS(4)

//* DELETE
DELETE 'HLQ.VSAM.KSDS' CLUSTER
```

### 4.5 Catalogs

| Catalog type | Purpose |
|-------------|---------|
| Master Catalog | One per z/OS system; points to user catalogs |
| User Catalog (ICF) | Contains dataset/VSAM entries by HLQ |
| Alias | Maps HLQ → user catalog |
| GDG Base | Generation Data Group definition |

```
DEFINE ALIAS(NAME('NEWHLQ') RELATE('UCAT.PROD'))

DEFINE GDG(NAME('HLQ.MYGDG') LIMIT(30) SCRATCH NOEMPTY)
```

### 4.6 GDG (Generation Data Groups)

| Reference | Meaning |
|-----------|---------|
| `HLQ.GDG(0)` | Current (most recent) generation |
| `HLQ.GDG(+1)` | New generation (create) |
| `HLQ.GDG(-1)` | Previous generation |
| `HLQ.GDG(-2)` | Two generations back |

Absolute name: `HLQ.GDG.G0045V00` — generation 45, version 0.

---

## 5. Communication Server — TCP/IP & VTAM (Vol 4)

### 5.1 TCP/IP Stack

z/OS Communications Server provides a full TCP/IP stack running in its own address space (TCPIP).

```
Applications (FTP, TN3270, HTTP, sshd)
       │
  Sockets API (BPX1xxx / C sockets)
       │
  TCP/IP Stack (TCPIP address space)
       │
  OSA-Express (network adapter)
       │
  Physical Network
```

Configuration datasets:
- `PROFILE.TCPIP` — TCP/IP configuration (ports, routing, interfaces)
- `TCPIP.DATA` — resolver config (domain, nameserver)
- `hlq.ETC.SERVICES` / `hlq.ETC.HOSTS` — like /etc/services, /etc/hosts

Key TCP/IP commands:
```
D TCPIP,TCPIP,NETSTAT,CONN      — active connections
D TCPIP,TCPIP,NETSTAT,STATS     — statistics
V TCPIP,TCPIP,STOP,PORT=8080    — stop a listener
D TCPIP,TCPIP,NETSTAT,ROUTE     — routing table
```

### 5.2 VTAM/SNA

VTAM manages SNA (Systems Network Architecture) — 3270 terminals, printers, LU6.2 sessions.

| VTAM Command | Purpose |
|--------------|---------|
| `D NET,ID=applname` | Display VTAM application status |
| `V NET,ACT,ID=nodename` | Activate a node/application |
| `V NET,INACT,ID=nodename` | Deactivate |
| `D NET,SESSIONS` | Show active sessions |
| `D NET,MAJNODES` | List major nodes |

TN3270 Server (provides 3270 access over TCP/IP):
```
Port 23 (telnet) or custom → TN3270 Server → VTAM LU → Application (TSO, CICS, IMS)
```

---

## 6. Parallel Sysplex (Vol 5)

### 6.1 Sysplex Architecture

```
┌─────────────────────────────────────────────────┐
│              Coupling Facility (CF)              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Lock    │  │ Cache   │  │ List    │        │
│  │Structure│  │Structure│  │Structure│        │
│  └─────────┘  └─────────┘  └─────────┘        │
└─────────┬────────────┬───────────┬──────────────┘
          │            │           │
     ┌────┴────┐  ┌────┴────┐  ┌──┴──────┐
     │ LPAR A  │  │ LPAR B  │  │ LPAR C  │
     │ (z/OS)  │  │ (z/OS)  │  │ (z/OS)  │
     └─────────┘  └─────────┘  └─────────┘
     CTC/CF links    Sysplex Timer    XCF
```

| Component | Purpose |
|-----------|---------|
| Coupling Facility (CF) | Shared cache, locks, lists across LPARs |
| XCF (Cross-system Coupling Facility) | Inter-system communication |
| GRS (Global Resource Serialization) | Enqueue across systems |
| Sysplex Timer | Synchronized TOD clock |
| ARM (Automatic Restart Manager) | Restart failed work on another LPAR |
| System Logger | Shared log streams |
| RRS (Resource Recovery Services) | Two-phase commit across resources |

### 6.2 GDPS (Geographically Dispersed Parallel Sysplex)

Disaster recovery with automatic site failover:
- Metro Mirror (synchronous copy) — 0 data loss, ~300km max
- Global Mirror (asynchronous) — RPO seconds/minutes, unlimited distance
- HyperSwap — automatic DASD volume swap on failure

---

## 7. Security — RACF (Vol 6)

### 7.1 RACF Architecture

RACF (Resource Access Control Facility) controls ALL access on z/OS.

```
User requests resource → SAF (System Authorization Facility)
  → RACF database check
    → User profile (RACF segment)
    → Group membership
    → Resource profile (DATASET, FACILITY, PROGRAM, etc.)
    → Access level (NONE / READ / UPDATE / CONTROL / ALTER)
  → Permit or deny
```

### 7.2 RACF Key Commands

| Command | Purpose |
|---------|---------|
| `LU userid ALL` | List user profile (all segments) |
| `LG groupname` | List group members |
| `LISTDSD DA('dsn') ALL` | List dataset profile |
| `RLIST class profile ALL` | List general resource profile |
| `PE 'dsn' ID(user) ACCESS(READ)` | Permit dataset access |
| `PERMIT profile CLASS(class) ID(user) ACCESS(lvl)` | Permit general resource |
| `ALU userid PASSWORD(pass)` | Reset password |
| `ALU userid RESUME` | Resume revoked user |
| `SEARCH CLASS(USER) REVOKE` | Find revoked users |
| `SETROPTS RACLIST(class) REFRESH` | Refresh in-storage profiles |

### 7.3 RACF Classes (most common)

| Class | Protects |
|-------|----------|
| DATASET | Datasets (generic/discrete profiles) |
| FACILITY | General facilities (started tasks, USS, etc.) |
| PROGRAM | Program access control |
| SURROGAT | Surrogate submit authority |
| STARTED | Started task userid assignment |
| UNIXPRIV | USS superuser privileges |
| CSFKEYS | Cryptographic keys (ICSF) |
| OPERCMDS | Operator commands |

### 7.4 USS Security

| Resource | How secured |
|----------|-------------|
| UID/GID assignment | `ALU user OMVS(UID(nnn) HOME('/home/user') PROGRAM('/bin/sh'))` |
| File permissions | Standard POSIX (rwxrwxrwx) + RACF ACLs |
| Superuser | UID=0 or UNIXPRIV class (BPX.SUPERUSER) |
| Daemon authority | BPX.DAEMON facility class |
| ptrace | BPX.DEBUG facility class |

---

## 8. Problem Diagnosis (Vol 8)

### 8.1 Problem Classification

| Indicator | Type | First action |
|-----------|------|--------------|
| Sxxx abend | System abend | Check abend code table |
| Uxxx abend | User abend | Check application documentation |
| Wait state | System hang | Check wait state code on HMC |
| Loop | Infinite loop | `D A,jobname` → check CPU consumption |
| IEA995I | SVC dump taken | Find dump in SYS1.DUMPxx |
| IEATDUMP | Transaction dump | Check SYSMDUMP/SYSUDUMP DD |

### 8.2 System Abend Quick Reference

| Abend | Meaning | Key diagnostic |
|-------|---------|----------------|
| S001 | I/O error | Check IEC message, run IEHLIST |
| S013 | DCB conflict | Compare LRECL/BLKSIZE: DD vs dataset |
| S0C1 | Operation exception | Invalid instruction — check load module |
| S0C4 | Protection exception | Bad pointer/subscript — check CEEDUMP |
| S0C7 | Data exception | Non-numeric in packed field — check input |
| S0CB | Divide by zero | Check divisor field |
| S106/S806 | Module not found | Verify STEPLIB/JOBLIB/LNKLST |
| S122/S222 | Cancelled/TIME | TIME exceeded or operator cancel |
| S213 | Dataset not found on volume | Catalog vs VTOC mismatch |
| S322 | Time limit | Increase TIME or fix loop |
| S878/S80A | Virtual storage | Increase REGION |
| SB37/SD37/SE37 | Out of space | Add secondary space or reallocate |
| S913 | RACF access denied | Check PERMIT authority |

### 8.3 Diagnostic Data Sources

| Source | What it contains | How to get it |
|--------|-----------------|---------------|
| SYSLOG | All WTO messages from system | SDSF LOG or OPERLOG |
| LOGREC | Hardware/software errors | EREP utility (IFCEREP1) |
| SYS1.DUMPxx | SVC dumps | IPCS to browse |
| SYSMDUMP/SYSUDUMP | Transaction dumps | DD in JCL |
| GTF Trace | Generalized Trace Facility | START GTF, then IPCS |
| SMF Records | Performance/accounting | SYS1.MANx datasets |
| CEEDUMP | Language Environment dump | Auto-generated at LE abend |
| CTRACE | Component trace | `TRACE CT,ON,COMP=xxx` |

### 8.4 IPCS (Interactive Problem Control System)

IPCS is the dump analysis tool — accessed via ISPF option or TSO.

| IPCS Command | Purpose |
|--------------|---------|
| `SETDEF` | Set defaults (DSNAME, ASID) |
| `STATUS` | Abend summary |
| `SUMMARY` | Quick problem overview |
| `WHERE` | Show PSW at time of error |
| `REGS` | Display registers |
| `SYSTRACE` | System trace entries |
| `VERBX LEDATA` | Language Environment analysis |
| `VERBEXIT LOGDATA` | LOGREC analysis |
| `LIST addr LEN(x) ASID(xx)` | Display storage |
| `FIND pattern` | Search dump for hex/char pattern |
| `CBF` | Control Block Formatter |

### 8.5 Trace Processing

| Trace type | Captures | Start command |
|-----------|----------|---------------|
| GTF (Generalized Trace) | SVC, I/O, PROG, DSP events | `S GTF.GTF,MEMBER=GTFPRMxx` |
| CTRACE (Component Trace) | Specific component internals | `TRACE CT,ON,COMP=SYSJES2` |
| System Trace | Always-on; last N events per CPU | In dump (SYSTRACE in IPCS) |
| Master Trace | Last 4000 WTOs | `D T` |

---

## 9. z/OS UNIX System Services (Vol 9)

### 9.1 USS Architecture

z/OS UNIX = full POSIX-compliant UNIX environment running within z/OS.

```
┌─────────────────────────────────────────────┐
│ z/OS UNIX Applications                      │
│ (shell, utilities, servers, custom apps)    │
├─────────────────────────────────────────────┤
│ z/OS UNIX Kernel Services (BPXPRMxx)        │
│ - Process management (fork/exec/wait)       │
│ - File system interface (open/read/write)   │
│ - Signals, pipes, sockets                   │
│ - IPC (shared memory, semaphores, MQs)      │
├─────────────────────────────────────────────┤
│ Physical File Systems                       │
│ - zFS (z File System) — default             │
│ - TFS (Temp File System) — /tmp             │
│ - NFS (Network File System) — client/server │
│ - automount                                 │
├─────────────────────────────────────────────┤
│ z/OS Base Services (MVS)                    │
│ - WLM, RACF, RMF, SMF                      │
└─────────────────────────────────────────────┘
```

### 9.2 Shell and Utilities

Default shells: `/bin/sh` (z/OS shell), Rocket ports may add `bash`, `zsh`.

| Command | z/OS-specific notes |
|---------|---------------------|
| `ls -la` | Shows file tags in `ls -T` |
| `chtag -p file` | Display file tag (CCSID) |
| `chtag -t -c IBM-1047 file` | Tag as EBCDIC |
| `chtag -t -c ISO8859-1 file` | Tag as ASCII |
| `iconv -f IBM-1047 -t ISO8859-1` | Convert EBCDIC↔ASCII |
| `cp "//'HLQ.PDS(MBR)'" ./file` | Copy MVS dataset → USS |
| `cp ./file "//'HLQ.PDS(MBR)'"` | Copy USS → MVS dataset |
| `tsocmd "LISTDS 'HLQ.DS'"` | Run TSO command from USS |
| `mvscmd --pgm=IEFBR14 --sysprint=stdout` | Run MVS program from USS |
| `submit "//'HLQ.JCL(JOB)'"` | Submit JCL from USS |
| `opercmd "D A,L"` | Issue operator command |
| `df -k /path` | Filesystem usage |
| `mount` | Show mount points |

### 9.3 File System Administration

```bash
# Create a zFS filesystem
zfsadm define -aggregate OMVS.USER.ZFS -volumes VOL001 -cylinders 100 50

# Mount
/usr/sbin/mount -f OMVS.USER.ZFS -t ZFS /u/users/newdir

# Grow a zFS
zfsadm grow -aggregate OMVS.USER.ZFS -size 200

# Check/repair
zfsadm fsinfo -aggregate OMVS.USER.ZFS
```

### 9.4 BPXPRMxx Parameters

Key parmlib settings for USS:

| Parameter | Purpose | Example |
|-----------|---------|---------|
| MAXPROCSYS | Max processes system-wide | 65536 |
| MAXPROCUSER | Max processes per user | 512 |
| MAXFILEPROC | Max open files per process | 65536 |
| MAXTHREADS | Max threads per process | 4096 |
| MAXCORESIZE | Max core dump size | 4194304 |
| ROOT | Root filesystem mount | OMVS.ROOT.ZFS |
| MOUNT | Additional mount points | |
| FILESYSTYPE | Register file system types | ZFS, TFS, NFS |
| AUTOCVT | Auto-conversion ON/OFF/ALL | |

### 9.5 Environment Variables (z/OS-specific)

| Variable | Purpose |
|----------|---------|
| `_BPXK_AUTOCVT` | Enable auto-conversion (ON/OFF/ALL) |
| `_CEE_RUNOPTS` | Language Environment runtime options |
| `_BPX_SHAREAS` | Share address space (YES/NO/REUSE) |
| `_BPX_JOBNAME` | Set MVS job name for USS process |
| `_TAG_REDIR_IN/OUT/ERR` | Auto-tag redirected I/O |
| `IBM_JAVA_OPTIONS` | JVM options |
| `LIBPATH` | Shared library search path (like LD_LIBRARY_PATH) |
| `STEPLIB` | MVS load library from USS |
| `PATH` | Must include `/bin:/usr/bin` minimum |

### 9.6 OMVS Command (TSO to USS)

```
OMVS                    — Enter USS shell from TSO
OMVS EXEC('/bin/ls')    — One-shot command
ISHELL                  — ISPF-based file manager for USS
BPXBATCH SH 'cmd'      — Run USS command in batch (JCL)
```

Batch JCL for USS:
```jcl
//USSCMD   EXEC PGM=BPXBATCH
//STDIN    DD DUMMY
//STDOUT   DD SYSOUT=*
//STDERR   DD SYSOUT=*
//STDPARM  DD *
SH cd /u/users/mydir && ls -la && df -k .
/*
```

---

## 10. Performance Management — WLM, RMF, SMF (Vol 11, 12)

### 10.1 WLM (Workload Manager)

WLM manages system resources to meet service-level goals.

```
Service Definition:
  ├── Service Class (performance goals)
  │     ├── Execution velocity goal (e.g., 80%)
  │     ├── Response time goal (e.g., <200ms for 90%)
  │     └── Discretionary (best-effort)
  ├── Classification Rules (which work → which class)
  │     └── Match on: subsystem, transaction name, userid, program, etc.
  └── Resource Groups (optional caps)
```

| WLM Command | Purpose |
|-------------|---------|
| `D WLM,SCHENV=*` | Display scheduling environments |
| `D WLM,APPLENV=*` | Display application environments |
| `V WLM,POLICY=name,REFRESH` | Activate/refresh policy |
| `D WLM,RESOURCE=*` | Resource status |

### 10.2 RMF (Resource Measurement Facility)

RMF collects performance data.

| RMF Report | Data |
|------------|------|
| Monitor I | Interval reports (CPU, storage, I/O, paging) |
| Monitor II | Real-time data (address space, delay) |
| Monitor III | Sysplex-wide (WLM service, CF activity) |
| Postprocessor | Historical reports from SMF 70-79 |

### 10.3 SMF (System Management Facilities)

SMF records everything for accounting, performance, security audit.

| SMF Record | Content |
|------------|---------|
| Type 4 | Step termination |
| Type 5 | Job termination |
| Type 14/15 | Dataset open (input/output) |
| Type 30 | Common address space work |
| Type 42 | SMS statistics |
| Type 70-79 | RMF data |
| Type 80 | RACF security events |
| Type 89 | RACF processing |
| Type 101 | DB2 accounting |
| Type 110 | CICS statistics |
| Type 119 | TCP/IP statistics |

### 10.4 Performance Indicators

| Metric | Healthy | Action if bad |
|--------|---------|---------------|
| CPU utilization (CP) | <85% | Consider upgrade or zIIP offload |
| Paging rate | <10 pages/sec | Add real storage |
| CSA usage | <80% of max | Check for storage leaks |
| Spool usage | <70% | Purge old output |
| Enqueue contention | Minimal | Check GRS, SYSIEFSD (JES spool) |
| Channel busy | <50% | Spread I/O across paths |
| Response time | Within SLA | WLM goal adjustment |

---

## 11. REXX Programming

### 11.1 REXX Template (TSO)

```rexx
/* REXX - Standard template */
TRACE OFF
SIGNAL ON ERROR
SIGNAL ON SYNTAX

/* Parse arguments */
ARG dataset_name option

IF dataset_name = '' THEN DO
  SAY 'Usage: MYEXEC dataset_name [option]'
  EXIT 8
END

/* Read a dataset */
"ALLOC F(INPUT) DA('"dataset_name"') SHR REUSE"
IF RC <> 0 THEN DO
  SAY 'ERROR: Cannot allocate' dataset_name 'RC='RC
  EXIT 12
END

"EXECIO * DISKR INPUT (STEM rec. FINIS"
"FREE F(INPUT)"

SAY 'Read' rec.0 'records from' dataset_name

/* Process records */
count = 0
DO i = 1 TO rec.0
  IF POS('ERROR', rec.i) > 0 THEN DO
    count = count + 1
    SAY 'Line' i':' STRIP(rec.i)
  END
END

SAY count 'error lines found.'
EXIT 0

ERROR:
  SAY 'ERROR at line' SIGL': RC='RC
  EXIT 16
SYNTAX:
  SAY 'SYNTAX error at line' SIGL':' ERRORTEXT(RC)
  EXIT 16
```

### 11.2 REXX with ISPF Services

```rexx
/* REXX - ISPF member list */
ADDRESS ISPEXEC

dsn = "'HLQ.SOURCE.PDS'"
"LMINIT DATAID(did) DATASET("dsn") ENQ(SHR)"
IF RC <> 0 THEN EXIT RC
"LMOPEN DATAID("did") OPTION(INPUT)"

member = ''
DO FOREVER
  "LMMLIST DATAID("did") OPTION(LIST) MEMBER(member) STATS(YES)"
  IF RC <> 0 THEN LEAVE
  SAY LEFT(member,8) ZLCDATE ZLMDATE ZLUSER
END

"LMCLOSE DATAID("did")"
"LMFREE DATAID("did")"
EXIT 0
```

### 11.3 REXX Built-in Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `SUBSTR(s,n,len)` | Substring | `SUBSTR('HELLO',2,3)` → `'ELL'` |
| `POS(needle,hay)` | Find position | `POS('X','ABXYZ')` → `3` |
| `STRIP(s,'B')` | Remove spaces | Both/Leading/Trailing |
| `LEFT(s,n)` / `RIGHT(s,n)` | Pad/truncate | |
| `COPIES(s,n)` | Repeat string | `COPIES('-',72)` |
| `TRANSLATE(s,out,in)` | Character translation | |
| `DATATYPE(s,'N')` | Check if numeric | Returns 1/0 |
| `WORD(s,n)` / `WORDS(s)` | Word extraction | |
| `DATE('S')` | Sorted date (YYYYMMDD) | |
| `TIME('N')` | Normal time (HH:MM:SS) | |

---

## 12. Batch Processing Patterns

### 12.1 Standard Batch Job Template

```jcl
//BATCHJOB JOB (ACCT),'BATCH PROCESS',
//         CLASS=A,MSGCLASS=X,MSGLEVEL=(1,1),
//         REGION=0M,TIME=NOLIMIT,NOTIFY=&SYSUID
//*================================================================
//* STEP 1: DELETE OLD OUTPUT (IGNORE NOT-FOUND)
//*================================================================
//DELETE   EXEC PGM=IDCAMS
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  DELETE 'HLQ.OUTPUT.FILE' NONVSAM
  SET MAXCC=0
/*
//*================================================================
//* STEP 2: SORT INPUT
//*================================================================
//SORT     EXEC PGM=SORT
//SORTIN   DD DSN=HLQ.INPUT.FILE,DISP=SHR
//SORTOUT  DD DSN=HLQ.SORTED.FILE,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(50,25),RLSE),
//            DCB=(RECFM=FB,LRECL=200,BLKSIZE=0)
//SYSOUT   DD SYSOUT=*
//SYSIN    DD *
  SORT FIELDS=(1,10,CH,A,15,8,PD,D)
  INCLUDE COND=(25,2,CH,EQ,C'TX')
/*
//*================================================================
//* STEP 3: PROCESS
//*================================================================
// IF (SORT.RC <= 4) THEN
//PROCESS  EXEC PGM=MYPROG,REGION=512M
//STEPLIB  DD DSN=HLQ.LOADLIB,DISP=SHR
//INPUT    DD DSN=HLQ.SORTED.FILE,DISP=SHR
//OUTPUT   DD DSN=HLQ.OUTPUT.FILE,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(100,50),RLSE),
//            DCB=(RECFM=VB,LRECL=32756,BLKSIZE=32760)
//SYSPRINT DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
// ENDIF
```

### 12.2 DB2 Batch Pattern

```jcl
//DB2STEP  EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD DSN=DB2.SDSNLOAD,DISP=SHR
//         DD DSN=HLQ.LOADLIB,DISP=SHR
//SYSTSPRT DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
//SYSTSIN  DD *
  DSN SYSTEM(DB2P)
  RUN PROGRAM(MYPROG) PLAN(MYPLAN) -
      LIB('HLQ.LOADLIB')
  END
/*
```

### 12.3 CICS Batch (DFHCSDUP, CEDA)

```jcl
//CSDUPD   EXEC PGM=DFHCSDUP
//STEPLIB  DD DSN=CICS.SDFHLOAD,DISP=SHR
//DFHCSD   DD DSN=CICS.CSD,DISP=SHR
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  DEFINE PROGRAM(NEWPROG) GROUP(MYGROUP)
         LANGUAGE(COBOL) DATALOCATION(ANY)
  DEFINE TRANSACTION(NTRN) GROUP(MYGROUP)
         PROGRAM(NEWPROG) TASKDATALOC(ANY)
  ADD GROUP(MYGROUP) LIST(MYLIST)
/*
```

---

## 13. Common Operator Commands

| Command | Purpose |
|---------|---------|
| `D A,L` | Display all active address spaces |
| `D A,jobname` | Display specific job |
| `D T` | Display date/time |
| `D M` | Display system status |
| `D GRS,CONTENTION` | Show enqueue contention |
| `D GRS,RES=(qname,rname)` | Who holds a resource |
| `D SMS,STORGRP(ALL),LISTVOL` | SMS storage group utilization |
| `D U,VOL=volser` | Display volume info |
| `D ASM` | Display paging activity |
| `D PROG,LPA` | Display LPA status |
| `D PROG,LNKLST` | Display LNKLST concatenation |
| `D PROG,APF` | Display APF list |
| `S procname` | Start a started task |
| `P procname` | Stop a started task |
| `C jobname` | Cancel job |
| `F procname,command` | Modify (send command to STC) |
| `V (device),ONLINE/OFFLINE` | Vary device |
| `SETPROG APF,ADD,DSN=x,VOL=y` | Add APF entry dynamically |

---

## 14. Key Message Prefixes

| Prefix | Component | Area |
|--------|-----------|------|
| IEA | Supervisor | Nucleus, abend processing |
| IEB | Data management | Utilities (IEBCOPY, etc.) |
| IEC | Data management | I/O errors, OPEN/CLOSE |
| IEE | Master scheduler | Console, commands |
| IEF | Job management | Allocation, scheduling |
| IGD | SMS/DFSMS | Storage management |
| ICH | RACF | Security |
| CSV | Contents Supervision | LPA, LNKLST, APF |
| IKJ | TSO/E | TSO command processing |
| IST | VTAM | Network communication |
| EZB/EZD | TCP/IP | Communications Server |
| BPX | USS | Unix System Services |
| DFHAC/SM/FC | CICS | CICS components |
| DSN | DB2 | Database |
| IOS | I/O Supervisor | Channel paths, devices |
| IAR | RSM | Real Storage Manager |
| IRA | SRM/WLM | System Resources Manager |

---

## 15. Quick Troubleshooting Decision Tree

```
Problem reported
├── Job failed?
│   ├── JCL ERROR → Check IEFC messages (syntax)
│   ├── Sxxx abend → See abend table (§8.2)
│   ├── Uxxx abend → Check application docs/CEEDUMP
│   ├── RC > 4 → Check SYSPRINT for error messages
│   └── Not running → Check $D J, D A, class, priority
├── Dataset problem?
│   ├── Not found → LISTCAT, check catalog/alias
│   ├── In use → D GRS,RES=(SYSDSN,dsn)
│   ├── Out of space → SB37/SD37 → ADRDSSU DEFRAG or reallocate
│   └── VSAM error → Check IDCAMS LISTCAT, VERIFY
├── Security denied?
│   ├── S913 → LU user, LISTDSD DA('dsn') ALL
│   ├── ICH messages → RLIST class profile ALL
│   └── USS EPERM → Check UID/GID, file perms, UNIXPRIV
├── Performance?
│   ├── CPU bound → WLM service class, optimize code
│   ├── I/O bound → Check channel paths, spread data
│   ├── Storage bound → REGION, CSA, paging
│   └── Contention → GRS enqueue, DB2 locks
└── Network?
    ├── Connection refused → D TCPIP,NETSTAT,CONN, PORT check
    ├── VTAM inactive → D NET,ID=appl, V NET,ACT
    └── DNS/routing → TCPIP.DATA, PROFILE.TCPIP ROUTE
```
