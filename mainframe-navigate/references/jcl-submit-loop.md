# JCL Submit End-to-End Loop

Reliable pattern to build → submit → poll → fetch spool → parse RC → return structured result. Uses MCP tools where possible, with REXX fallback for SDSF drill.

---

## Standard jobcard

```
//<userid>X JOB (SAF,P1D),'CMM HC PROBE',
//         CLASS=J,
//         MSGCLASS=X,
//         MSGLEVEL=(1,1),
//         NOTIFY=&SYSUID,
//         REGION=0M,
//         TIME=NOLIMIT
```

Rules:
- **Job name** = 1–8 chars, must match `<userid>` prefix for site RACF policy (JES2 `JOBCLASS(x) PROT` and `JESJOBS SUBMIT.<node>.<userid>.<jobname>`). Convention: `<userid>` + 1 letter suffix identifying phase (`C`=CSV, `D`=DB2, `S`=SMF).
- **CLASS** = agreed batch class (usually `J` or `A`). Site policy.
- **MSGCLASS** = output class held for you to browse (`X`).
- **NOTIFY=&SYSUID** = send `IEF404I` on completion to your TSO.
- **REGION=0M** = maximum below-the-bar storage. Above-the-bar handled by `MEMLIMIT` (usually parmlib default).
- **TIME=NOLIMIT** = no CPU time cap. Use `TIME=(minutes,seconds)` in production.
- **TYPRUN=SCAN** = JCL syntax-check only, no execution. Use for validation.
- **TYPRUN=HOLD** = wait for `A` (release) in SDSF ST.

---

## The 6-step loop

### 1. Build the JCL

Write to a temp PDS member. USS staging + `cp -T` to PDS is the pattern:

```bash
# In OMVS (or from bash via mcp__tn3270-bridge__mainframe_send in row 21)
cat > /tmp/probe.jcl <<'EOF'
//C30T158X JOB (SAF,P1D),'PROBE',CLASS=J,MSGCLASS=X,
//         NOTIFY=&SYSUID,REGION=0M,TIME=NOLIMIT
//STEP1    EXEC PGM=IEFBR14
//DUMMY    DD DUMMY
EOF
chtag -tc ISO8859-1 /tmp/probe.jcl
cp -T /tmp/probe.jcl "//'C30T158.CMM.JCL(PROBE)'"
```

The `cp -T` converts ASCII → EBCDIC and honors PDS record format (FB/80). Chtag first to prevent AUTOCVT double-conversion.

Alternative: `mainframe_write_dataset dsn=C30T158.CMM.JCL member=PROBE content=[...]` (v2.0+ MCP tool, guarded).

### 2. Validate syntax (optional but recommended)

Change `CLASS=J` to `CLASS=J,TYPRUN=SCAN` and submit. SDSF will show CC=0000 for valid JCL, non-zero with IEFC codes for errors. Then flip TYPRUN off and resubmit.

### 3. Submit

```
mainframe_submit_job dsn=C30T158.CMM.JCL member=PROBE
```

Response includes `IKJ56250I JOB C30T158X(JOB12345) SUBMITTED`. Parse the JOBID.

Alternative from OMVS:
```
submit "//'C30T158.CMM.JCL(PROBE)'"
```

Alternative from TSO:
```
tsocmd "SUBMIT 'C30T158.CMM.JCL(PROBE)'"
```

### 4. Poll for completion

Fastest: `TSO STATUS <jobid>`:

```bash
tsocmd "STATUS C30T158X(JOB12345)"
```

Response evolves:
- `IKJ56211I JOB C30T158X(JOB12345) EXECUTING` — running
- `IKJ56211I JOB C30T158X(JOB12345) ON OUTPUT QUEUE` — done, output available
- `IKJ56212I JOB C30T158X(JOB12345) NOT FOUND` — purged or bad JOBID

Loop:
```rexx
/* REXX */
address TSO
_jobid = "JOB12345"
_jobname = "C30T158X"
_deadline = time("s") + 300
do until pos("ON OUTPUT QUEUE", _out) > 0 | time("s") > _deadline
  "STATUS " _jobname "(" _jobid ")" outtrap("_out.")
  call syscalls "SLEEP 5"
end
```

Or SDSF ST panel:
```
mainframe_send fieldName=COMMAND text="ST" aid=ENTER
mainframe_send fieldName=COMMAND text="PREFIX C30T158X" aid=ENTER
mainframe_send fieldName=COMMAND text="ST" aid=ENTER
```

### 5. Fetch spool output

Programmatic path — REXX `OUTPUT` in a batch TSO:

```rexx
"OUTPUT " _jobname "(" _jobid ") PRINT('C30T158.JESLOG.PROBE')"
```

Interactive path — SDSF drill:
```
mainframe_send fieldName=COMMAND text="ST" aid=ENTER
# find job row, position cursor col 1
mainframe_cursor row=<row> col=1
mainframe_send text="?" aid=ENTER
# now on DDNAME list; select JESMSGLG
mainframe_cursor row=<jesmsglg_row> col=1
mainframe_send text="SB" aid=ENTER
# now in ISPF Browse; use mainframe_read_all to grab everything
mainframe_read_all scrollKey=PF8 stopText="BOTTOM OF DATA"
```

Or export via `XDC`:
```
mainframe_cursor row=<jesmsglg_row> col=1
mainframe_send text="XDC" aid=ENTER
mainframe_send labelLeft="Dataset name" text="C30T158.EXPORT.JESMSGLG" aid=ENTER
mainframe_read_dataset dsn=C30T158.EXPORT.JESMSGLG
```

### 6. Parse RC

JESMSGLG contains one `IEF142I` line per step:
```
IEF142I C30T158X STEP1 - STEP WAS EXECUTED - COND CODE 0000
```

And one job-level line:
```
IEF404I C30T158X - ENDED - TIME=15.32.11
```

For ABEND:
```
IEF450I C30T158X STEP1 - ABEND=S806 U0000 REASON=00000004
```

Regex:
- Step CC: `IEF142I\s+\S+\s+(\S+)\s+-\s+STEP WAS EXECUTED\s+-\s+COND CODE\s+(\d+)`
- ABEND: `IEF450I\s+\S+\s+(\S+)\s+-\s+ABEND=([SU]\d{4})\s+U(\d{4})\s+REASON=(\S+)`
- Job end: `IEF404I\s+(\S+)\s+-\s+ENDED`

---

## Common RACF denials during submit

Every one of these appears in JESMSGLG and can be detected with a single grep:

| Message | Cause | Resolution |
|---|---|---|
| `ICH408I USER(uid) NOT AUTHORIZED TO xxx` | Dataset/resource read | `PE 'dsn' CLASS(DATASET) ID(uid) ACCESS(READ)` |
| `IEF238D … REPLY 'RETRY' OR 'CANCEL'` | Dataset in use | Wait or find holder (SDSF ENQ) |
| `IEC150I 913-38, IFG0194F` | Dataset security violation | Same as ICH408I |
| `IEF257I ALLOC FOR MEMBER NOT AUTHORIZED` | PDS member access | RACF DATASET class |
| `IKJ56709I INSUFFICIENT AUTHORITY` | TSO command auth | TSO segment + AUTH TSO |
| `IEF453I JOB JCL ERROR` | Bad JCL — see IEFC codes | Fix JCL |

---

## Common IEFC JCL errors

| Code | Cause |
|---|---|
| `IEFC001I` | JCL error preventing job scheduling |
| `IEFC005I` | Procedure not found (JCLLIB / SYSPROC missing) |
| `IEFC006I` | Symbolic parameter undefined |
| `IEFC019I` | Missing quote or paren |
| `IEFC078I` | ENDCNTL missing / control-statement error |
| `IEFC605I` | UNIDENTIFIED OPERATION FIELD |
| `IEFC609I` | KEYWORD ambiguous |
| `IEFC623I` | Symbol invalid |
| `IEFC648I` | Duplicate DDNAME |

---

## Anti-patterns

- **Fixed sleeps** (`sleep 30`) — brittle. Poll STATUS or use `mainframe_wait`.
- **Hard-coded JOBIDs** — parse from submit response.
- **Fetching by absolute output-queue position** — jobs slide when others finish. Use JOBNAME/JOBID filter.
- **Forgetting `TIME=NOLIMIT`** in interactive testing — S322 (CPU time exceeded).
- **PDS FB/80 with lines > 72 chars** — silent truncation. REXX/JCL must fit cols 1–72.
- **Submitting from a PDS you don't own** — `IKJ56250I` may succeed but job runs under owner's HLQ context, not yours.
- **Not chtag'ing before `cp -T`** — AUTOCVT can double-convert on binary uploads.
