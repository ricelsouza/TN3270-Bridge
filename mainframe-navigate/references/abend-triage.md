# ABEND Triage Playbook via TN3270

An ABEND is z/OS's way of saying "your program blew up." Every ABEND has a system code (S-codes like `S0C4`) or user code (U-codes like `U0016`). This reference is the field playbook for triaging common ABENDs via the TN3270 bridge — from initial detection in SDSF ST to root-cause fetch of CEEDUMP/SYSUDUMP.

For dump-level analysis (control-block walking, PSW/regs decode), see [`ipcs-basics.md`](ipcs-basics.md).

---

## Discovery flow

```
1. mainframe_send fieldName=COMMAND text="ST" aid=ENTER
2. mainframe_send fieldName=COMMAND text="PREFIX <userid>*" aid=ENTER
3. mainframe_send fieldName=COMMAND text="OWNER *" aid=ENTER
4. mainframe_send fieldName=COMMAND text="ST" aid=ENTER
5. Look for CC column values NOT starting with "CC 0000"
```

CC column formats:
| Display | Meaning |
|---|---|
| `CC 0000` | Success |
| `CC 0004` | Warning (RC 4) |
| `CC 0008` | Error (RC 8) |
| `CC 0012` / `CC 0016` | Severe error |
| `ABENDSxxx` | System ABEND code Sxxx |
| `ABENDUnnnn` | User ABEND code Unnnn |
| `JCL ERR` | JCL syntax error (never ran) |
| `SEC` | Security violation (RACF) |
| `SYS FAIL` | System failure during job |
| `<blank>` | Still running |

Position cursor on the job row, drill in with `?` or `SB`:

```
mainframe_cursor row=<jobrow> col=1
mainframe_send text="?" aid=ENTER            # DDNAME list
# On JESMSGLG row:
mainframe_cursor row=<jesmsglg_row> col=1
mainframe_send text="SB" aid=ENTER
mainframe_read_all scrollKey=PF8 stopText="BOTTOM OF DATA"
```

---

## Read JESMSGLG for the ABEND line

Every ABEND emits an `IEF450I` line:

```
IEF450I <jobname> <stepname> - ABEND=<code> U<usercode> REASON=<reason>
IEF450I EAE976B STEP1 - ABEND=S806 U0000 REASON=00000004
IEF450I EAE976B STEP1 - ABEND=S0C4 U0000 REASON=00000011
IEF450I EAE976B STEP1 - ABEND=U0016 U0016 REASON=00000000
```

Grep for `IEF450I`. Also look for:
- `IEF404I` — job ended (with time)
- `IEF142I` — step return code
- `IEF285I` — dataset kept/deleted at step end
- `IEA995I` — symptom dump summary
- `IEA989I` — SVC dump captured (with dump title)
- `CEE3204S` etc. — LE runtime message

---

## Common ABEND codes

### `S0C4` — protection exception

**Cause**: program tried to read/write memory it doesn't own. Classic pointer bug (garbage pointer, uninitialized).

**Reason code hints**:
- `0000000B` — page fault, page not in real storage AND ineligible
- `00000010` — protection exception, key mismatch
- `00000011` — segment translation exception (bad DAT)

**Look for**:
- `CEEDUMP` in DDNAMEs — LE dump has traceback, TCB chain, register save area
- `SYSUDUMP` — MVS unformatted dump
- PSW right-hand word = failing address

**Common fixes**:
- Uninitialized COBOL pointer (OCCURS DEPENDING ON, LINKAGE)
- C wild pointer / off-by-one
- REXX passing invalid TCB
- Passed area shorter than program expects

### `S0C7` — data exception

**Cause**: bad decimal data (COMP-3 field with non-numeric bytes, PACK/UNPK failure).

**Look for**: CEEDUMP shows COBOL variable + hex dump around failing offset.

**Common fixes**:
- SPACES in a numeric field
- Uninitialized WORKING-STORAGE numeric
- Data from file with wrong offset

### `S806` — program not found

**Cause**: `EXEC PGM=xxx` where xxx not in STEPLIB / JOBLIB / LINKLIST / LPA.

**Look for**: `IEF212I` before ABEND — `STEP dsn NOT FOUND` or `DDNAME=STEPLIB DATA SET NOT FOUND`.

**Common fixes**:
- Missing STEPLIB DD or wrong DSN
- Program in a library not in LNKLST/LPA and no STEPLIB
- Program name typo (case-sensitive on some sites for USS-linked programs)

### `S013` — dataset open failure

**Reason codes**:
- `00000014` — LRECL mismatch (DD LRECL ≠ dataset LRECL)
- `00000018` — RECFM mismatch
- `00000020` — BLKSIZE mismatch
- `00000024` — DSORG mismatch
- `0000005C` / `0000005D` — attempt to open a PDS as PS or vice versa
- `00000060` — member not found in PDS

**Look for**: `IEC141I` message near the ABEND — describes exact incompatibility.

**Common fixes**:
- DD statement DCB attributes ≠ dataset attributes
- Wrong DISP (`DISP=OLD` when concurrent SHR needed)
- Member not there

### `S222` — cancelled by operator

Job was cancelled with `C jobname`. Usually intentional.

### `SB37` / `SD37` / `SE37` — space abend

**Cause**: out of space on a dataset.
- `SB37` — no space on volume for secondary extent
- `SD37` — primary allocation exhausted, no secondary defined
- `SE37` — max extents (16 for non-SMS, 255 for SMS-managed) reached

**Look for**: `IEC030I` / `IEC031I` / `IEC032I` messages.

**Common fixes**:
- Increase `SPACE=(CYL,(prim,sec))` primary and secondary
- Add secondary if only primary defined
- For SMS: check DATACLAS extent limits, consider PDSE
- Compress PDS (IEBCOPY) to reclaim deleted-member space

### `S913` — RACF violation during OPEN

**Cause**: user tried to open a dataset without READ/UPDATE authority.

**Look for**: `ICH408I USER(uid) DATASET(dsn) VIOLATION` right before ABEND.

**Common fixes**:
- Grant RACF DATASET access (needs security team confirmation)
- Use a different HLQ the user owns
- Add user to owning group

### `S878` — insufficient virtual storage

**Cause**: subpool 229/230 exhaustion, or requested GETMAIN below-the-bar too large.

**Look for**: `IEA602I` or `IEA766I` around the failure. `REGION=0M` may still hit sub-pool limits.

**Common fixes**:
- Increase REGION
- Use above-the-bar (64-bit) storage
- LE `HEAP` runtime options
- Check MEMLIMIT

### `S322` — CPU time exceeded

**Cause**: job ran longer than `TIME=(minutes,seconds)`.

**Fix**: `TIME=NOLIMIT` in jobcard, or fix runaway logic.

### `S806-04` — module referenced but not authorized

Program is APF-only but running in a non-APF address space.

**Fix**: run under a batch initiator class that inherits APF, or add STEPLIB library to APF list (needs sysprog confirmation).

### `U0016` (COBOL) — signals + exceptions

COBOL LE termination. Look at CEEDUMP for CEE-msg (`CEE3204S`, `CEE3213S`).

### `U4038` (LE) — signal received without handler

Program received an unhandled signal (typically SIGSEGV translated).

### `U3999` — CEE runtime error

Generic LE failure. See `IGZMSG` / `CEE-msg` for detail.

---

## When JESMSGLG isn't enough — CEEDUMP

If the DDNAME list has `CEEDUMP`, browse it (`SB` line command). Contents:

- **Job info** — LE version, options, storage report
- **Traceback** — call chain with entry names + statement numbers
- **Registers** — GPR + FPR at ABEND
- **PSW** — Program Status Word
- **Storage report** — heap + stack usage per level
- **Data dump** — WORKING-STORAGE and LINKAGE for the failing routine (compile-time option: `MAP,LIST`)

Traceback line format:
```
CEE3INF <Entry> +hex_offset in load module <name> at address <hex>
```

Trace UPWARD from bottom to find your source line.

---

## When only SYSMDUMP available — IPCS

SYSMDUMP is machine-readable, meant for `IPCS`. See [`ipcs-basics.md`](ipcs-basics.md).

---

## Automation pattern for the SHC

For `phase_smf` post-processing:

1. Read SMF Type 30 subtype 5 records (step-level).
2. Detect non-zero completion codes.
3. Cross-reference with SMF Type 30 subtype 4 (job-level) — same JCT_ID.
4. If step ABEND: capture jobname + stepname + code + timestamp.
5. Correlate with SMF Type 42 (DFSMS) if space-related.
6. Emit finding: `"Job X abended with Sxxx in step Y at HH:MM:SS"`.

Never re-run the failed job automatically — always let the user decide.

---

## Anti-patterns

- **Assuming CC = SUCCESS** — CC 4 or 8 is often ignored but hides warnings. Set jobs to fail on CC > 4 in critical batches.
- **Skipping JESMSGLG in favor of SYSPRINT** — JESMSGLG has the definitive `IEF450I`. SYSPRINT might have application-level messages, not the OS truth.
- **Deleting jobs in SDSF ST before browsing** — `P` purges the spool. Once purged, IPCS can only work if SYSMDUMP was captured to a persistent dataset (unusual by default).
- **`SYSUDUMP` in a job that produces `SYSMDUMP` also** — pick one. Both waste spool.
- **`ABEND=S806 REASON=04` mistaken for missing library** — REASON=04 = "not authorized" (APF), not "not found." REASON=00 or 08 = not found.
