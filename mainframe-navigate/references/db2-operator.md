# DB2 z/OS Operator Commands Reference (Read-Only)

DB2 for z/OS exposes a rich `-` (dash-prefix) operator command set. This reference is READ-ONLY. Any command that modifies state — `-START/-STOP DATABASE`, `-BIND`, `-ALTER`, `-TERM UTILITY`, `-CANCEL THREAD`, `-RECOVER`, `-ACCESS DATABASE ... MODE(FORCE)`, catalog DDL/DML — is **out of scope** for this skill and requires explicit user confirmation.

`-DSN` prefix (subsystem name) prepends every command. Examples below use `-DB2P`; substitute your SSID (`ALGT` on ACW2/SPS1 test).

DB2 commands can be issued from:
- **SDSF ULOG** with `/` prefix: `/-DB2P DIS DATABASE(*)`
- **TSO** via DSN command processor
- **SDSF LOG** (system console commands)
- **JCL** via `DSNTEP2` (SQL) or `DSNTIAR`
- **Batch REXX** via `DSNREXX` + `ADDRESS DSNREXX "EXECSQL …"`

---

## Threads and connections

### `-DIS THREAD` — active threads

```
-DB2P DIS THREAD(*)                                 # all
-DB2P DIS THREAD(*) TYPE(ACTIVE)                    # only active
-DB2P DIS THREAD(*) TYPE(INDOUBT)                   # 2PC in-doubt
-DB2P DIS THREAD(*) DETAIL                          # verbose
-DB2P DIS THREAD(*) CONN(SERVER)                    # by connection type
-DB2P DIS THREAD(*) LOCATION(*) DETAIL              # for DDF
```

Fields:
- `TOKEN` — DB2-internal thread ID
- `NAME` — jobname or DDF client
- `PLAN`/`PACKAGE` — running plan or package
- `TYPE` — connection type
- `TCPU/ECPU/QCPU` — CPU consumption
- `SUSPEND` — current wait state
- `V447-CURRENT` — active SQL statement / cursor

### `-DIS DDF` — Distributed Data Facility

```
-DB2P DIS DDF DETAIL
```

Reports:
- `STATUS` (`STARTD`, `STOPD`, `SUSPND`)
- `LOCATION`
- `IPADDR`, `TCPPORT`, `RESPORT`
- `SQL CONNS`, `ACTIVE`, `MAXCONNS`, `MAXDBAT`, `CONDBAT`

### `-DIS LOCATION` — remote catalogs

```
-DB2P DIS LOCATION DETAIL
```

Every location DB2 has heard about (from CDB or DDF). Useful for federated setups.

---

## Databases & tablespaces

### `-DIS DATABASE`

```
-DB2P DIS DB(*)                                     # every DB
-DB2P DIS DB(DB01) SPACENAM(*)                      # all TS + IX under one DB
-DB2P DIS DB(DB01) SPACENAM(TS*)                    # tablespace only
-DB2P DIS DB(DB01) SPACENAM(*) LIMIT(*)             # remove default 50 limit
-DB2P DIS DB(DB01) SPACENAM(*) RESTRICT             # only restricted states
-DB2P DIS DB(DB01) SPACENAM(*) USE                  # what's using each object
-DB2P DIS DB(DB01) SPACENAM(*) LOCKS                # locks held
-DB2P DIS DB(DB01) SPACENAM(*) CLAIMERS             # claimers (long-running readers)
-DB2P DIS DB(DB01) SPACENAM(*) LPL                  # logical page list
-DB2P DIS DB(DB01) SPACENAM(*) WEPR                 # write-error pages
```

Status codes (in the `STATUS` column):
- `RW` — read-write (normal)
- `RO` — read-only
- `STOP` — stopped
- `UT` — utility in progress
- `UTRW/UTRO/UTUT` — utility-imposed state
- `RECP` — recovery pending
- `CHKP` — check pending
- `RESTP` — restart pending
- `WEPR` — write-error pending
- `LPL` — logical page list (page(s) unavailable)
- `GRECP` — group buffer pool recover pending
- `LSTOP` — logical stop
- `AREO*` — advisory reorg pending (RTS)
- `AREST` — advisory restart

**Any status other than `RW` deserves attention.** `RESTP`/`RECP`/`LPL`/`GRECP` require SME intervention.

---

## Buffer pools

### `-DIS BUFFERPOOL`

```
-DB2P DIS BUFFERPOOL(*)                             # summary
-DB2P DIS BUFFERPOOL(BP0)                           # one pool
-DB2P DIS BUFFERPOOL(BP0) DETAIL                    # allocation
-DB2P DIS BUFFERPOOL(BP0) DETAIL(*)                 # verbose stats
-DB2P DIS BUFFERPOOL(BP0) LSTATS                    # interval delta since last query
-DB2P DIS BUFFERPOOL(BP0) LIST(*)                   # objects in this pool
-DB2P DIS BUFFERPOOL(BP0) GBPDEP                    # coupling facility usage
```

Metrics to watch:
- `VPSIZE` / `HPSIZE` — virtual/hiperspace pages
- `VPSEQT` — sequential-steal threshold (%)
- `DWQT` — deferred write threshold
- `PGSTEAL` — steal algorithm (LRU / FIFO / NONE)
- `GETPAGE` — total requests
- `SYNC READS` — synchronous DASD reads (bad if high)
- `PREFETCH READS` — sequential/list/dynamic prefetch
- `PAGE-INS REQ` — real-storage pageins for buffer
- `HIT RATIO` — derived (getpage - reads) / getpage; target > 95%

### `-DIS BUFFERPOOL(*) GBPDEP`

Coupling Facility group buffer pool dependency (data sharing). Reports GBP structure use.

---

## Utilities

### `-DIS UTILITY`

```
-DB2P DIS UTIL(*)                                   # all in-progress
-DB2P DIS UTIL(UTILID)                              # specific utility
```

Fields per utility:
- `UTILID` — assigned utility ID
- `STATEMENT` — current phase (UTILINIT/RELOAD/BUILD/SORTBLD/…)
- `PHASE` — sub-phase
- `COUNT` — records/pages processed
- `STATUS` — `ACTIVE`, `STOPPED`, `TERM`

Mutating (out of scope): `-TERM UTILITY(UTILID)`, `-START UTILITY`.

---

## Log & recovery

### `-DIS LOG`

```
-DB2P DIS LOG
```

Reports:
- `COPY1/COPY2` — active log dataset names
- `TSTAMP OF EARLIEST/LATEST` records
- `CHECKPOINT FREQUENCY`
- `TOTAL LOG WRITES`
- `WRITES SUSPENDED`

`WRITES SUSPENDED > 0` indicates DB2 waited on log space — investigate archiving.

### `-DIS ARCHIVE`

```
-DB2P DIS ARCHIVE
```

Archive log status — allocation, tape/DASD, log range.

---

## Groups (data sharing)

### `-DIS GROUP`

```
-DB2P DIS GROUP
-DB2P DIS GROUP DETAIL
```

For data-sharing groups: all members, their DB2 release level, CF structure use, and IRLM state. Non-data-sharing: reports single member.

---

## Traces (read-only inspection; mutation requires confirmation)

### `-DIS TRACE`

```
-DB2P DIS TRACE
-DB2P DIS TRACE(*)
-DB2P DIS TRACE(*) DEST(*)
-DB2P DIS TRACE(ACCTG,STAT,MON) DEST(*) COMMENT('active')
```

Trace types: `ACCTG` (SMF 101), `STAT` (SMF 100), `AUDIT`, `MON` (Monitor), `PERFM` (Performance), `GLOBAL`.

Mutating (out of scope): `-START TRACE`, `-STOP TRACE`, `-MODIFY TRACE`.

---

## Statistics

### `-DIS STATS`

```
-DB2P DIS STATS
```

Rarely used interactively; standard consumers pull SMF 100/101 records.

---

## Plans & packages

### `-DIS PLAN`

```
-DB2P DIS PLAN                                       # active plans
-DB2P DIS PLAN(PLANNAME) SHOWCOL                     # plan detail
```

BIND-related (out of scope for read-only): `-BIND`, `-REBIND`, `-FREE`.

---

## Catalog SELECT patterns

Read-only catalog queries via `DSNTEP2` batch:

```jcl
//DSNTEP2  EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD DISP=SHR,DSN=DB2P.SDSNLOAD
//         DD DISP=SHR,DSN=DB2P.RUNLIB.LOAD
//SYSTSPRT DD SYSOUT=*
//SYSPRINT DD SYSOUT=*
//SYSTSIN  DD *
 DSN SYSTEM(DB2P)
 RUN PROGRAM(DSNTEP2) PLAN(DSNTEP2) PARMS('ALIGN(MID)')
 END
/*
//SYSIN    DD *
  SELECT DBNAME, NAME, NACTIVE, SPACE, STATUS
  FROM SYSIBM.SYSTABLESPACE
  WHERE DBNAME LIKE 'CMM%'
  ORDER BY DBNAME, NAME
  ;
/*
```

**Rule**: only `SELECT` — no `INSERT/UPDATE/DELETE/CREATE/ALTER/DROP/GRANT/REVOKE`.

Key catalog tables (all under `SYSIBM.`):
- `SYSTABLES` — tables + views + aliases
- `SYSCOLUMNS` — columns
- `SYSTABLESPACE` — tablespace metadata + status
- `SYSINDEXES` — indexes
- `SYSINDEXPART` — index partitions
- `SYSTABLEPART` — tablespace partitions
- `SYSTABLESPACESTATS` — RTS (V10+, requires RTS active)
- `SYSINDEXSPACESTATS` — RTS
- `SYSPACKAGE` — packages
- `SYSPLAN` — plans
- `SYSDATABASE` — databases
- `SYSSTOGROUP` — storage groups
- `SYSCOPY` — image copy history
- `SYSDBAUTH` / `SYSTABAUTH` / `SYSPACKAUTH` — GRANT records
- `SYSENVIRONMENT` (V12+ NFM) — zparm values
- `SYSDUMMY1` — 1-row 1-column dummy
- `SYSROUTINES` — stored procedures + UDFs

Get DB2 version:
```sql
SELECT GETVARIABLE('SYSIBM.VERSION') AS DB2VER FROM SYSIBM.SYSDUMMY1
```

---

## Common patterns

### Find long-running threads
```
-DB2P DIS THREAD(*) TYPE(ACTIVE) DETAIL
```
Look at `TCPU/ECPU` — high CPU with no progress = candidate for cancel (but ask user first).

### Find tablespaces in trouble
```
-DB2P DIS DB(*) SPACENAM(*) RESTRICT
```
Every tablespace in non-normal state.

### Find utilities stuck
```
-DB2P DIS UTIL(*)
```
Look for `STATUS=STOPPED` with recent PHASE — usually a log-space or DASD issue.

### Buffer pool health check
```
-DB2P DIS BUFFERPOOL(*) DETAIL(*) LSTATS
```
`SYNC READS` high vs `GETPAGE` = pool too small. `PAGE-INS REQ > 0` = pool paged out (real storage constraint).

---

## Anti-patterns

- **Interpreting `-DIS DB(*) SPACENAM(*)` without `LIMIT(*)`** — default caps to first 50, misleading.
- **Running `-DIS THREAD(*) DETAIL` in busy prod** — output can be 1000+ lines; use filters.
- **`-DIS TRACE` interpreted as "trace is off"** — TRACE could be class-active but DEST is wrong. Check DEST(*).
- **Assuming `RTS` is populated** — requires `STATSTIME` to advance; `SYSTABLESPACESTATS.STATSTIME` may be all-zeros on new sites.
- **Running catalog SELECT with `LIKE 'X%'`** without column index — full-scan. Add `FETCH FIRST N ROWS ONLY`.
