# RACF Read-Only Inspection Reference

RACF (Resource Access Control Facility) is z/OS's mandatory access control. This reference covers ONLY read-only commands for security posture inspection. Mutation commands are listed at the end as a **blocklist** — this skill never issues them without explicit user confirmation.

---

## Golden rule

**Never issue any of these without user confirmation:**

- `PE` / `PERMIT` — grant/revoke access
- `ALU` / `ALTUSER` — modify user
- `AU` / `ADDUSER` — create user
- `DU` / `DELUSER` — remove user
- `AG` / `ADDGROUP` — create group
- `DG` / `DELGROUP` — remove group
- `CO` / `CONNECT` — add user to group
- `RE` / `REMOVE` — remove user from group
- `RDEF` / `RDEFINE` — create/modify resource profile
- `RDEL` / `RDELETE` — delete resource profile
- `RALT` / `RALTER` — alter resource profile
- `SETR` / `SETROPTS` — global settings
- `ADDSD` — add dataset profile
- `DELDSD` — delete dataset profile
- `ALTDSD` — alter dataset profile
- `PWSYNC`, `PASSWORD`, `PHRASE` mutations
- `RVARY` — deactivate RACF (extreme)
- `RRSF` remote sharing changes

**Rule of thumb**: if a command's first two letters are `PE`, `RD`, `AL`, `AD`, `SE`, or `RV` and it operates on RACF classes, treat as mutation and confirm.

---

## Read-only inspection commands

### `LU` / `LISTUSER` — user info

```
LU C30T158
LU C30T158 OMVS                  # z/OS UNIX segment
LU C30T158 TSO                   # TSO segment
LU C30T158 CICS                  # CICS segment
LU C30T158 NETVIEW               # NetView segment
LU C30T158 ALL                   # all segments
LU C30T158 CSDATA                # customer segment
```

Output includes:
- `USER=C30T158` — RACF userid
- `NAME=` — display name
- `OWNER=` — who owns this profile
- `CREATED=` — creation date
- `DEFAULT-GROUP=` — primary group
- `PASSDATE=` / `PASS-INTERVAL=` — password state
- `LAST-ACCESS=` — last login
- `CLASS AUTHORIZATIONS=` — RDEFINE allowed classes
- `INSTALLATION-DATA=` — free-form site notes
- `GROUP=<name> AUTH=USE|CREATE|CONNECT|JOIN` — connected groups + permission level
- `ATTRIBUTES=` — SPECIAL / OPERATIONS / AUDITOR / GRPACC / ADSP / REVOKE / PROTECTED

**Attribute flags to watch**:
- `SPECIAL` — global admin (can PE anything)
- `OPERATIONS` — bypass DATASET/tape access checks
- `AUDITOR` — read all profiles + audit records
- `REVOKE` — account disabled
- `PROTECTED` — cannot log in interactively
- `RESTRICTED` — no default access (only explicit PE)

OMVS segment fields: `UID`, `HOME`, `PROGRAM`, `CPUTIMEMAX`, `ASSIZEMAX`, `FILEPROCMAX`, `PROCUSERMAX`, `THREADSMAX`, `MMAPAREAMAX`.

### `LG` / `LISTGRP` — group info

```
LG GRPACQ
LG GRPACQ OMVS                   # z/OS UNIX segment (GID)
LG GRPACQ CSDATA
```

Output:
- `SUPERIOR GROUP=` — parent in the group hierarchy
- `OWNER=` — profile owner
- `CREATED=`
- `INSTALLATION-DATA=`
- `SUBGROUP(S)=` — child groups
- `USER(S)=` with `AUTH=` level for each (`USE` most common, `CREATE/CONNECT/JOIN` administrative)

OMVS segment: `GID` (POSIX numeric group id).

### `LISTDSD` — dataset profile

```
LISTDSD DA('SYS1.PARMLIB')
LISTDSD DA('SYS1.PARMLIB') ALL
LISTDSD PREFIX('C30T158') ALL         # all profiles under prefix
LISTDSD DA('SYS1.PARMLIB') GENERIC    # generic profile (with %/**)
LISTDSD DA('SYS1.PARMLIB') AUTHUSER   # who is on the access list
LISTDSD DA('SYS1.PARMLIB') HISTORY    # last audit records
```

Output:
- `LEVEL=` — profile level (0=discrete, 1=generic)
- `OWNER=`
- `UNIVERSAL ACCESS=` (`NONE` / `READ` / `UPDATE` / `CONTROL` / `ALTER` / `EXECUTE`)
- `WARNING=YES/NO` — if YES, violations logged but not denied (dangerous)
- `ERASE=YES/NO` — physical erase on delete
- `NOTIFY=userid` — get email on violation
- `AUDITING=` — audit level per access
- `USER ACCESS-LIST` — who + at what level
- `INSTALLATION DATA=`

### `RLIST` — resource profile (non-DATASET classes)

```
RLIST FACILITY BPX.SUPERUSER
RLIST FACILITY BPX.**
RLIST FACILITY IRR.DIGTCERT.* AUTHUSER
RLIST OPERCMDS MVS.DISPLAY.** ALL
RLIST DSNR DB2P.**
RLIST SDSF ISFCMD.ODSP.STATUS.**
RLIST JESJOBS SUBMIT.*.C30T158.**
RLIST JESSPOOL <node>.C30T158.**.**
RLIST TSOAUTH ACCT
RLIST TSOPROC ISPFPROC
RLIST APPL <appl-id>
RLIST STARTED **
RLIST PROGRAM ** ALL
RLIST XFACILIT * AUTHUSER
```

Common classes to know:
| Class | What it protects |
|---|---|
| `DATASET` | Datasets (use LISTDSD, not RLIST) |
| `FACILITY` | Named facilities (BPX.*, IRR.*, HZS.*, MOUNT.*) |
| `OPERCMDS` | Console commands (MVS.DISPLAY.*, MVS.MODIFY.*, MVS.CANCEL.*) |
| `DSNR` | DB2 subsystem access (DB2SSID.*) |
| `SDSF` | SDSF panels/functions (ISFCMD.*, ISFAUTH.*) |
| `JESJOBS` | Job submission (SUBMIT.node.owner.jobname) |
| `JESSPOOL` | Spool datasets (node.owner.jobname.*) |
| `TSOAUTH` | TSO authorities (ACCT, MOUNT, OPER) |
| `TSOPROC` | Which TSO PROC user can use |
| `APPL` | ISPF/CICS/APPL access |
| `STARTED` | Started task security |
| `PROGRAM` | Program pathing (APF authorized) |
| `TERMINAL` | 3270 terminal access |
| `LOGSTRM` | Log stream access |
| `SURROGAT` | Surrogate submit (submit as another user) |
| `XFACILIT` | Extended facilities (site-defined) |

### `SEARCH` — find profiles

```
SEARCH CLASS(FACILITY) FILTER(BPX.**)
SEARCH CLASS(DATASET) FILTER(C30T158.**)
SEARCH CLASS(DATASET) USER(C30T158)                # datasets this user has access to
SEARCH CLASS(FACILITY) NOMASK                      # non-wildcarded profiles
SEARCH CLASS(DATASET) WARNING                      # profiles with WARNING=YES (audit)
SEARCH CLASS(FACILITY) NOTIFY(userid)              # profiles that notify this userid
```

### `LISTGRP *` — all groups you can see

```
LISTGRP *
```

Only groups where you have `AUDITOR` or `SPECIAL` are fully visible; others show minimal info.

### `IRRUT200` / `IRRUT400` — RACF database utilities (batch)

```jcl
//IRRUT200 EXEC PGM=IRRUT200,PARM='INDEX'
//SYSPRINT DD SYSOUT=*
//SYSRACF  DD DISP=SHR,DSN=SYS1.RACF
```

Batch dump/verify RACF database. Read-only. Requires READ on `SYS1.RACF`.

### RACF certificate inventory (read-only)

```
RACDCERT LIST                                        # personal certs
RACDCERT ID(CICSA) LIST                              # certs owned by an ID
RACDCERT CERTAUTH LIST                               # CA certs
RACDCERT LISTRING(*) ID(CICSA)                       # keyring inventory
```

---

## Health-check questions to answer

When asked "audit RACF posture for X":

1. **Who has SPECIAL/OPERATIONS?** `SEARCH CLASS(USER) FILTER(**) ATTR(SPECIAL,OPERATIONS)`
2. **Are there WARNING datasets?** `SEARCH CLASS(DATASET) WARNING`
3. **UACC READ on sensitive HLQs?** `LISTDSD DA('SYS1.PARMLIB')` and check UACC.
4. **PROTECTED userids?** `SEARCH CLASS(USER) FILTER(**) ATTR(PROTECTED)`
5. **REVOKED but still exists?** `SEARCH CLASS(USER) FILTER(**) ATTR(REVOKE)`
6. **Password-expired accounts still active?** `SEARCH CLASS(USER) FILTER(**) PWDAYS(GT 90)`
7. **Groups with too many members?** `LG <name>` and count `USER(S)`.
8. **Surrogate abuse?** `SEARCH CLASS(SURROGAT) FILTER(**)`
9. **APF library changes?** `RLIST PROGRAM ** ALL` + cross-reference with `D PROG,APF`.
10. **BPX facility grants (root access)?** `RLIST FACILITY BPX.SUPERUSER AUTHUSER`

---

## Anti-patterns

- **Assuming NOACC when profile is missing** — RACF's default depends on class + SETROPTS. Check `SETROPTS LIST` (needs AUDITOR/SPECIAL).
- **Generic profile FILTER without `NOMASK`** — you get every match including wildcards. Add `NOMASK` for discrete only.
- **`LISTDSD PREFIX` on a large HLQ** — output is enormous. Use `AUTHUSER` to focus on ACL.
- **Interpreting `WARNING=YES` as safe** — it's NOT. WARNING logs violations but grants access. Site should have zero WARNING profiles in production.
- **Reading LU output as authoritative** — check `SETROPTS LIST PASSWORD` for site policy that overrides per-user.
