# z/OS Console Command Matrix

MVS console commands are issued via:
- **SDSF ULOG** with `/` prefix (site-specific `OPERCMDS` MCSOPER READ)
- **TSO** via `CONSOLE` command (requires `TSOAUTH OPER`)
- **JCL** via `MODIFY` addressing a system-level console
- **MCS console** directly (operator terminal)

This reference covers **read-only DISPLAY commands** in depth. Mutating verbs (`F`, `S`, `P`, `C`, `K`, `V`, `Z`, `SLIP`, `SETXCF`, `SETPROG`, `SETOMVS`, `SETSMF`) are listed as a **blocklist** — never issue without explicit user confirmation.

---

## The DISPLAY (`D`) verb — read-only

`D` shows system state. Every `D` command is safe. Categories:

### System identity + IPL
```
D IPLINFO                            # IPL date/time, load parm, IEASYS suffix, sysres volume
D IPLINFO,SYSPLEX                    # sysplex info (name, members, CDSs)
D SYMBOLS                            # active system symbols (&SYSNAME., etc.)
D T                                  # local time + date
D CLOCK                              # STPOFF, ETR/STP state
D ETR                                # ETR (Sysplex Timer) status
D STP                                # STP (Server Time Protocol) status
D M=CPU                              # CP inventory (physical + logical + IIP/zIIP + SUP)
D M=STOR                             # storage inventory (real + expanded + increments)
D M=CHP                              # channel path status
D M=DEV(0A00-0A0F)                   # specific device range
D M=CONFIG                           # HCD config
```

### Address spaces / jobs
```
D A,ALL                              # ALL address spaces (JOBS/STC/TSU/SYS)
D A,L                                # summary line
D A,jobname                          # specific job
D A,STC                              # started tasks only
D A,TSU                              # TSO users
D J,jobname                          # detail one job (may fail on some sites)
```

### JES2 status
```
$D J(nnnn)                           # JES2 job details (needs `$` prefix)
$D I                                 # initiator status
$D Q                                 # queue depth
$D N,Q=X                             # network status
```

### GRS (global resource serialization)
```
D GRS                                # GRS mode + peers
D GRS,C                              # CONTENTION (essential for stuck-job triage)
D GRS,ANALYZE                        # deep contention analyze
D GRS,SCOPE=SYSTEM                   # local ENQs only
D GRS,SYSTEM=SYSNAME                 # specific member ENQs
```

### Paging / storage
```
D ASM                                # aux storage manager (page slots)
D ASM,PAGE                           # PAGE datasets in use
D ASM,SWAP                           # SWAP datasets
D VS=(volser,volser)                 # dataset activity on volume
D VIRTSTOR                           # virtual storage summary
```

### System Logger
```
D LOGGER,STATUS                      # coupling facility log streams state
D LOGGER,C                           # connections
D LOGGER,L,LSN=name                  # specific log stream
```

### WLM (Workload Manager)
```
D WLM                                # WLM mode + policy name
D WLM,APPLENV=*                      # application environments
D WLM,SYSTEMS                        # sysplex members + WLM mode per member
D WLM,POLICY                         # active service policy
D WLM,SC=srvcls                      # specific service class
D WLM,RC=rptcls                      # specific report class
```

### DFSMS
```
D SMS                                # SMS active mode
D SMS,STORGRP(ALL)                   # all storage groups
D SMS,STORGRP(ALL),LISTVOL           # + volumes per storage group
D SMS,CACHE                          # cache statistics
D SMS,VOLUME(volser)                 # specific volume
D SMS,OAM,ACTIVE                     # OAM state (object storage)
```

### XCF / Coupling Facility (Sysplex)
```
D XCF                                # basic sysplex status
D XCF,S                              # systems in sysplex
D XCF,C                              # coupling channels
D XCF,COUPLE                         # couple datasets
D XCF,STR                            # structures (CF resources)
D XCF,STR,STRNAME=name               # detail one structure
D XCF,STR,STATUS=REBUILDING          # only structures rebuilding
D XCF,CF                             # coupling facilities
D XCF,POLICY,TYPE=CFRM               # CFRM policy
D XCF,POLICY,TYPE=SFM                # SFM (Sysplex Failure Management)
D XCF,POLICY,TYPE=ARM                # ARM (Automatic Restart Manager)
D XCF,POLICY,TYPE=LOGR               # log stream policy
D XCF,ARMSTATUS                      # ARM element state
```

### I/O
```
D IOS,CONFIG                         # IOCDS + dynamic activate history
D IOS,MIH                            # MIH values per DASD class
D U,DASD,ONLINE,,64                  # online DASD units (first 64)
D U,DASD,ALLOC                       # allocated devices
D U,,ALLOC,jobname                   # devices allocated to a job
D IOS,MIH,DEV=xxxx                   # specific device
```

### TCP/IP
```
D TCPIP                              # all TCP/IP stacks
D TCPIP,tcpipname,NETSTAT,HOME       # IP addresses on this stack
D TCPIP,tcpipname,NETSTAT,DEVLINKS   # interfaces (OSA, HiperSockets, IUTIQDIO)
D TCPIP,tcpipname,NETSTAT,STATS      # IP/TCP/UDP/ICMP counters
D TCPIP,tcpipname,NETSTAT,ALL        # active connections
D TCPIP,tcpipname,NETSTAT,ROUTE      # routing table
D TCPIP,tcpipname,NETSTAT,CONFIG     # stack configuration
D TCPIP,tcpipname,PROFILE            # PROFILE.TCPIP settings
D TCPIP,tcpipname,DEVLINK            # (compat)
```

### USS / OMVS
```
D OMVS                               # OMVS status
D OMVS,A=ALL                         # all USS address spaces
D OMVS,F                             # mounted file systems
D OMVS,O                             # active BPXPRMxx options
D OMVS,W                             # WAITING for services
D OMVS,PID=nnnn                      # specific process
D OMVS,U=userid                      # by user
D OMVS,L                             # limits (BPXPRMxx MAX*)
```

### SMF
```
D SMF,O                              # SYS1.MAN* datasets + recording + INTVAL
D SMF,S                              # status
D SMF,SUBSYS=name                    # subsystem SMF options
```

### RMF
```
D RMF                                # RMF started tasks
```

### Product inventory
```
D PROD                               # registered products
D PROD,STATE                         # + state (ENABLED/OWNED)
D PROD,REGISTER                      # registration events
D PROD,REL=z/OS.*                    # by product
```

### Program / library
```
D PROG,APF                           # APF-authorized libraries
D PROG,LNKLST                        # LNKLST concatenation
D PROG,LPA                           # LPA modules (from LPALSTxx + dynamic)
D PROG,EXIT                          # dynamic exits
D PROG,EXIT,EN=exit                  # specific exit + its exit routines
D PROG,LIBRARY=libname               # specific LPA library
D LLA                                # LLA status
```

### Parmlib
```
D PARMLIB                            # active PARMLIB concatenation + members used
```

### Dumps
```
D DUMP                               # dump datasets
D DUMP,STATUS                        # allocations + free
D DUMP,TITLE                         # SVC dump titles
D DUMP,OPTIONS                       # CHNGDUMP/SDUMPX options
```

### RRS (Resource Recovery Services)
```
D RRS
D RRS,URSTATUS                       # unit-of-recovery status
D RRS,LOG                            # RRS log stream
```

### Consoles
```
D C                                  # all consoles
D C,L                                # summary list
D C,MASTER                           # master console
D CONSOLES,SUMMARY                   # config summary
```

### Cryptography
```
D ICSF                               # ICSF (Integrated Cryptographic Service Facility) status
D CRYPTO                             # crypto express card summary
```

### DB2 (from console — need OPERCMDS DSNR SSID.*)
```
-DB2P DIS DDF DETAIL
-DB2P DIS DB(*) SPACENAM(*)
-DB2P DIS BUFFERPOOL(*)
                                     # see references/db2-operator.md for depth
```

### Sysplex-wide
```
D XCF,COUPLE,TYPE=WLM                # WLM CDS
D XCF,COUPLE,TYPE=SYSPLEX            # sysplex CDS
D IPLINFO,SYSPLEX                    # sysplex context
```

---

## Mutation blocklist — commands to NEVER issue without explicit user confirmation

### Job/task control
- `C jobname` / `C jobname,A=asid` — **CANCEL**
- `C jobname,DUMP` — cancel with dump
- `FORCE jobname,ARM` — force ARM restart
- `FORCE jobname,QUICK` — force address space termination
- `P jobname` — **STOP** (started task shutdown)
- `S jobname` / `S procname` — **START**
- `MODIFY (F) jobname,...` — modify (send command to job)

### System state changes
- `V device,OFFLINE` / `V device,ONLINE` — vary device
- `V PATH,online/offline` — vary channel path
- `V NET,INACT,ID=name` — VTAM inactivate
- `V NET,ACT,ID=name` — VTAM activate
- `V xcf,SYSNAME,OFFLINE` — remove system from sysplex (destructive)
- `Z EOD` — **quiesce for shutdown** (extreme)
- `Z NET,QUICK` — force VTAM stop
- `Z ABEND` — abend the operator console

### Parmlib mutations
- `SETPROG APF,ADD|DELETE` — add/remove APF library
- `SETPROG LNKLST,DEFINE|UNDEFINE|ACTIVATE|DEACTIVATE`
- `SETPROG LPA,ADD|DELETE`
- `SETPROG EXIT,ADD|DELETE|MODIFY`
- `SET IEA=xx` — activate IEASYSxx
- `SET GRS=xx` — activate GRSCNFxx
- `SET SMF=xx` — activate SMFPRMxx
- `SET IOS=xx` — activate IECIOSxx
- `SET OMVS=xx` — activate BPXPRMxx
- `SET SMS=xx` — activate IGDSMSxx
- `SET LOGR=xx` — activate IXGCNFxx
- `SET WLM=xx` — activate IEAOPTxx / IWMPOL
- `SET ALLOC=xx` — activate ALLOCxx
- `SET DEVSUP=xx` — activate DEVSUPxx

### SMF
- `SETSMF SYS(TYPE(...))` — SMF recording changes
- `SS TYPE=n` — start/stop SMF type

### XCF / Coupling Facility
- `SETXCF START,POLICY=name,TYPE=CFRM/SFM/ARM/LOGR`
- `SETXCF STOP,POLICY=...`
- `SETXCF STOP,STR=name`
- `SETXCF FORCE,STR=name,CON=connector`
- `SETXCF PRSMPOLICY,...`
- `SETXCF START,REBUILD,STRNAME=name`
- `SETXCF START,ALTER,STRNAME=name`

### OMVS
- `F BPXOINIT,SHUTDOWN=FORKS` — kill user forks (extreme)
- `F BPXOINIT,RESTART=FORKS`
- `F BPXOINIT,SUPERKILL=pid`
- `SETOMVS RESET=(xx)` — reset BPXPRMxx parm
- `SETOMVS SYSCALL_COUNTS=YES/NO`

### Slip traps (dump/trace mutations)
- `SLIP SET,...`
- `SLIP MOD,ID=name,...`
- `SLIP DEL,ID=name`
- `SLIP DISABLE,ID=name`
- `SLIP ENABLE,ID=name`
- `SLIP RESET`

### Dumps
- `DUMP TITLE=(...),JOBNAME=(...)` — request SVC dump (mutating in that it consumes DASD)
- `DUMPDS ADD,DSN=...` / `DUMPDS DEL,DSN=...`
- `CHNGDUMP SET,SDUMP=...` — change dump options

### GRS
- `V GRS,SYS=name` — vary system out of GRS (extreme)

### Consoles
- `V CN(name),ONLINE|OFFLINE`
- `K S,DEL` — delete queued messages (usually safe but confirm)
- `K A,I` — flush operator console
- `K Q,L=n` — set queue length

### System reset / IPL
- Any command that leads to IPL/reset is out of scope.

---

## Command execution patterns

### From SDSF ULOG
```
mainframe_send fieldName=COMMAND text="ULOG" aid=ENTER
mainframe_send fieldName=COMMAND text="/D IPLINFO" aid=ENTER
```

Result appears in ULOG window. `mainframe_read_all` grabs multi-page output.

### From TSO CONSOLE
```
tsocmd "CONSOLE"                     # enter console mode
tsocmd "D IPLINFO"                   # command
tsocmd "END"                         # exit console mode
```

Alt: `CONSPROF` to set profile first. Needs `TSOAUTH OPER`.

### From JCL
```jcl
//STEP1    EXEC PGM=MGCRE               # or IEBGENER-derived console-write utility
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  D IPLINFO
/*
```

Or use `MODIFY` to a running system-level task that echoes commands.

### From MCS console
Physical operator terminal. Out of scope for automation.

---

## Anti-patterns

- **Assuming `/` prefix in ULOG works** — needs OPERCMDS MCSOPER. Check with `LU userid OPER` or run `/D T` and expect `IEE345I` if denied.
- **Interpreting no output as failure** — some commands write to the console but ULOG shows only OPERLOG stream. Check `LOG` panel with `FIND '<msg>'`.
- **Issuing `V device,OFFLINE` to check state** — that IS the mutation. Use `D U,DASD` to check status non-destructively.
- **`D M=CPU` in a partition without adequate authority** — response can be censored. Full detail needs `SYS.MVS.CONFIG` READ.
- **`SETXCF STOP,STR=...` "just to see"** — not readable. If you want status, `D XCF,STR,STRNAME=name` is the read-only equivalent.
