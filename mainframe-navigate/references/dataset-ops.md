# Dataset Operations Reference

Catalog lookups, allocation, copy/move, VSAM, GDG, and ISPF utilities. Every command is TSO / batch JCL; assumes you are at TSO READY (`===>`) or in ISPF Option 6.

**Read-only defaults.** Any command that mutates state is flagged. Never issue `IDCAMS DELETE`, `TSO DELETE`, `IEBGENER DISP=(NEW,CATLG)` overwriting a non-owned HLQ, or `RENAME` a shared dataset without user confirmation.

---

## LISTC / LISTCAT — the catalog is your friend

```
LISTC ENTRIES('SYS1.PARMLIB') ALL
LISTC LVL('C30T158') NAMES
LISTC LVL('C30T158') ALL
LISTC ENT('C30T158.CMM.JCL') ALL
LISTC ENT('C30T158.CMM.GDG(0)') GDG                # GDG base + generations
LISTC ENT('DB2P.DSNDBC.**') NONVSAM                # every VSAM cluster under HLQ
LISTC ENT('SYS1.SMF.MAN*') CATALOG ALL              # dataset physical location
```

Common `LISTC` output fields:
- `CREATION`, `EXPIRATION`, `RACF`, `SMSDATA` (`STORAGECLASS`, `MGMTCLASS`, `DATACLASS`)
- `HISTORY` (last backup/reference/update)
- `EXTENTS`, `ALLOCATION`, `SPACE-TYPE`
- For VSAM: `INDEX`, `DATA` components; `CI-SIZE`, `SHR-OPTNS`, `HI-U-RBA`

To search by pattern:
```
LISTC LVL('C30T158') CREATION(LE 20260101) NAMES     # created before Jan 2026
```

---

## LISTDS — quick metadata

Faster than LISTCAT for RECFM/LRECL/DSORG. Also lists PDS members:

```
LISTDS 'C30T158.CMM.JCL' MEMBERS
LISTDS 'C30T158.CMM.JCL' MEMBERS STATUS               # incl ISPF stats
LISTDS 'C30T158.CMM.JCL' HISTORY                      # incl SMS classes
```

Output columns: DSORG, RECFM, LRECL, BLKSIZE, VOLUME, referenced date.

---

## LISTA — current allocations

Shows datasets DYNAMICally allocated in your TSO session:

```
LISTA STATUS
LISTA HISTORY
```

Useful when a `FREE` or `ALLOC` seems ignored.

---

## ALLOC / FREE — TSO dynamic allocation

```
ALLOC DA('C30T158.NEW.PS') NEW CATALOG UNIT(SYSDA) SPACE(1,1) TRACKS RECFM(F,B) LRECL(80) BLKSIZE(0)
ALLOC DA('C30T158.NEW.PS') OLD REUSE                  # to modify allocation
FREE DA('C30T158.NEW.PS')
```

Attributes:
- `NEW` / `OLD` / `MOD` / `SHR`
- `CATALOG` / `KEEP` / `UNCATALOG` / `DELETE`
- `UNIT(SYSDA)` — device unit name (SYSDA, VIO, TAPE)
- `SPACE(prim,sec)` `TRACKS` / `CYLINDERS` / `BLOCKS`
- `RECFM(F,B)` / `V,B` / `F,B,A` (`A` = ANSI carriage control)
- `LRECL(80)` — logical record length
- `BLKSIZE(0)` = system-determined (recommended)
- `DIR(10)` — directory blocks for PDS
- `LIKE('other.ds')` — copy attributes from existing dataset
- `DSORG(PS)` / `PO` — sequential vs PDS (obsolete; use `DIR(n)` for PDS)
- `VOL(volser)` — specific volume (uses site ACS routines otherwise)

---

## IDCAMS — Access Method Services

Batch utility for VSAM + catalog operations.

```jcl
//IDCAMS   EXEC PGM=IDCAMS,REGION=0M
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  LISTCAT ENTRIES('C30T158.**') ALL
  PRINT INDATASET('C30T158.CONFIG') COUNT(50)
  REPRO INDATASET('C30T158.CMM.JCL(PROBE)') OUTDATASET('C30T158.EXPORT.PROBE')
/*
```

Common IDCAMS verbs:
| Verb | Effect | MUTATION? |
|---|---|---|
| `LISTCAT` | List catalog | No |
| `PRINT` | Dump dataset contents (char/hex/dump) | No |
| `REPRO` | Copy dataset (fast) | Yes on target |
| `DEFINE CLUSTER` | Create VSAM | Yes |
| `DEFINE ALTERNATEINDEX` | Create AIX | Yes |
| `DEFINE GDG` | Create GDG base | Yes |
| `DEFINE NONVSAM` | Catalog a non-VSAM dataset | Yes on catalog |
| `DEFINE ALIAS` | Create catalog alias | Yes |
| `DELETE` | Delete dataset | Yes — HIGH IMPACT |
| `DELETE ... PURGE` | Delete despite RETPD | Yes — HIGH IMPACT |
| `DELETE ... ERASE` | Delete + overwrite tracks | Yes — HIGH IMPACT |
| `ALTER` | Change attributes | Yes |
| `VERIFY` | Fix VSAM open/close inconsistency | Yes on VSAM |
| `EXAMINE` | Check VSAM structure integrity | No |
| `EXPORT` / `IMPORT` | VSAM backup/restore | Yes |
| `LISTCRA` / `LISTVTOC` | Catalog Recovery Area / VTOC listing | No |

**Guardrail**: `DELETE` in IDCAMS is one of the most destructive operations on z/OS. Never issue without explicit user confirmation AND read the dataset with LISTC first to be certain.

---

## IEBGENER — copy/print sequential

Simple sequential copy. Works for PS-to-PS and PS-to-PDS(member).

```jcl
//STEP1    EXEC PGM=IEBGENER
//SYSPRINT DD SYSOUT=*
//SYSUT1   DD DISP=SHR,DSN=C30T158.SOURCE.PS
//SYSUT2   DD DISP=(NEW,CATLG,DELETE),DSN=C30T158.TARGET.PS,
//            UNIT=SYSDA,SPACE=(TRK,(5,5),RLSE),
//            DCB=(RECFM=FB,LRECL=80,BLKSIZE=0)
//SYSIN    DD DUMMY
```

Overwrite existing target: `DISP=SHR` on SYSUT2 (member) or `DISP=OLD` on PS. Overwriting IS mutation — confirm.

---

## IEBCOPY — PDS/PDSE copy, compress, unload

```jcl
//STEP1    EXEC PGM=IEBCOPY,REGION=0M
//SYSPRINT DD SYSOUT=*
//INPDS    DD DISP=SHR,DSN=C30T158.CMM.REXX
//OUTPDS   DD DISP=(NEW,CATLG,DELETE),DSN=C30T158.CMM.REXX.NEW,
//            UNIT=SYSDA,SPACE=(TRK,(5,5,5)),
//            DCB=(RECFM=FB,LRECL=80,BLKSIZE=0)
//SYSIN    DD *
  COPY OUTDD=OUTPDS,INDD=((INPDS,R))
  SELECT MEMBER=((MBR1),(MBR2))              /* only these members */
/*
```

Verbs:
- `COPY OUTDD=x,INDD=((y,R))` — R = replace existing members with same name
- `COPY OUTDD=x,INDD=y` — no replace (skip dup members)
- `COMPRESS INDD=x` — compact PDS (frees deleted-member space) — MUTATION but safe
- `SELECT MEMBER=(a,b,c)` — only these
- `EXCLUDE MEMBER=(a,b)` — skip these

---

## IEBUPDTE — apply source updates (SMP/E flavor)

Rarely needed except for old-school source distribution. Modern shops use SCLM or git-on-USS.

---

## IEHLIST — VTOC / catalog listing

Batch alternative to `LISTC` when you need VTOC-level detail:

```jcl
//STEP1    EXEC PGM=IEHLIST
//SYSPRINT DD SYSOUT=*
//DD1      DD DISP=SHR,UNIT=SYSDA,VOL=SER=WORK01
//SYSIN    DD *
  LISTVTOC VOL=SYSDA=WORK01,DSNAME=C30T158.**
/*
```

---

## GDG — Generation Data Groups

Model: define once, then reference by relative generation.

```
                             /* DEFINE GDG BASE (via IDCAMS) */
DEFINE GDG (NAME(C30T158.MYGDG) LIMIT(10) NOEMPTY SCRATCH)

                             /* DEFINE MODEL (non-SMS) */
ALLOC DA('C30T158.MYGDG.MODEL') NEW CATALOG UNIT(SYSDA) SPACE(1,1) TRACKS -
      RECFM(F,B) LRECL(80) BLKSIZE(0)
```

Reference in JCL:
```
//IN   DD DISP=SHR,DSN=C30T158.MYGDG(0)      /* current generation */
//IN2  DD DISP=SHR,DSN=C30T158.MYGDG(-1)     /* previous */
//OUT  DD DISP=(NEW,CATLG,DELETE),DSN=C30T158.MYGDG(+1),
//        SPACE=(TRK,(5,5),RLSE),
//        DCB=(*.MODEL)                       /* copy DCB from model, non-SMS */
```

Attributes:
- `LIMIT(n)` — max concurrent generations (n=1..255)
- `EMPTY` / `NOEMPTY` — behavior when LIMIT reached
- `SCRATCH` / `NOSCRATCH` — physically delete rolled-off generations
- `LIFO` / `FIFO` — order semantics
- `EXTENDED` — over 255 generations (z/OS 2.1+)

Diagnose:
```
LISTC ENT('C30T158.MYGDG') GDG                      # base info + current gen list
LISTC ENT('C30T158.MYGDG.G0001V00') ALL             # a specific generation
```

---

## ISPF utilities (option 3.x)

Every 3.x item is a menu-driven wrapper around IDCAMS/IEBGENER/IEBCOPY.

| Panel | Purpose | MUTATION? |
|---|---|---|
| 3.1 Library | View PDS attributes + members | No |
| 3.2 Data Set | Allocate/Rename/Delete/Reset/Print/Compress | Yes on Del/Ren/Reset/Compress |
| 3.3 Move/Copy | Move/Copy datasets or members | Yes on target |
| 3.4 DSLIST | Dataset list by HLQ/mask; `S/B/V/E/D/R/M/C` line commands | Potentially |
| 3.5 Reset | Reset ISPF stats | Yes (stats only, not data) |
| 3.6 Hardcopy | Print via JES | No |
| 3.7 Download | Download to workstation (host-only ISPF variant) | No |
| 3.8 Outlist | JES output list | No |
| 3.11 Format | SDSF-lite from ISPF | No |
| 3.12 SUPERC | File comparison | No |
| 3.13 SUPERCE | Extended file comparison | No |
| 3.14 Search-For | Grep-like search across datasets | No |
| 3.15 Compare | Fast compare | No |
| 3.17 UDList | User dataset list | No |
| 3.4S | SDSF via panel | No |

3.4 line commands (on a dataset row):
- `S` — select (usually EDIT for members, browse for VSAM)
- `E` — Edit
- `V` — View
- `B` — Browse
- `M` — Member list (PDS)
- `X` — Print
- `D` — Delete — MUTATION
- `R newname` — Rename — MUTATION
- `C dsn` — Copy to — MUTATION on target
- `Z` — Compress PDS — MUTATION but safe
- `I` — Info (attributes)
- `=` — Repeat last command

---

## Dataset naming conventions on ACW2 / SPS1

| Prefix | Purpose |
|---|---|
| `SYS1.*` | System datasets (parmlib, proclib, linklib) — READ ONLY |
| `SYS2.*` | Site system extensions |
| `USER.CMM.*` / `<userid>.CMM.*` | User's CMM work |
| `DB2.DSND10.*` | DB2 subsystem load libs (system-owned) |
| `HZS.HZSPDATA` | Health Checker persistent data |
| `TWS.PROD.*` | TWS/OPC system datasets |
| `ESSSAF.P*D.*` | ACI CMM production data (GAF, PTLF, config) |
| `WORK.**` | Site scratch space |

**Rule**: this skill only writes under `<userid>.**` unless the user has explicitly authorized a specific HLQ.

---

## Anti-patterns

- **`DELETE` in IDCAMS without LISTC first** — irreversible on non-SMS-managed datasets without a recent HSM backup.
- **`REPRO` overwriting a live VSAM without SHR options check** — silent corruption of concurrent readers.
- **PDS instead of PDSE for source authoring** — no space reclamation on member delete. Prefer PDSE (`LIBRARY` type) for source.
- **`SPACE=(TRK,(1,1),RLSE)` for growing datasets** — one extent, quick B37 abend (out of space). Use CYL with generous secondary.
- **`DCB=(RECFM=FB,LRECL=80)` on VB source** — lines > 76 get lost. Confirm with LISTDS first.
- **Absolute GDG name (`.G0001V00`)** — brittle. Always use relative `(0)/(+1)/(-1)`.
