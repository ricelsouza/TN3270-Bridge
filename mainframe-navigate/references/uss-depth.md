# z/OS UNIX Depth Reference (OMVS / USS)

USS (Unix System Services) is z/OS's POSIX shell environment. This reference covers the parts most easily overlooked when driving USS through TN3270: file tags, encoding conversion, MVS bridging, ISHELL, and BPXBATCH from JCL.

---

## Enter USS

From ISPF Primary Menu:
```
mainframe_send fieldName=OPTION text="6" aid=ENTER
mainframe_send text="OMVS" aid=ENTER
```

Or from TSO READY:
```
tsocmd "OMVS"
```

Alternative: **ISHELL** (option 6 then `ISHELL`) — an ISPF-based file manager for USS. Safer for casual browse; less powerful than OMVS.

Exit OMVS:
```
mainframe_send text="exit" aid=ENTER
mainframe_send aid=ENTER   # confirm
```

---

## OMVS panel quirks

- Input line is `===>` at row 21 (24-row model). Send to row 21, col 7.
- Refresh scrolled output: **PF10** (NOT PF8).
- Prompt is `$` (or `#` for root). Prompt reappears after each command.
- `FSUM2364` warning = you typed into a partial-overlay screen; press **PA2** to refresh.
- `FSUM2361` = input from wrong row; use `mainframe_cursor row=21 col=7` first.
- Command length ≤ ~72 chars per 3270 input field. Longer commands: put in a script (`cat > /tmp/x.sh <<EOF ... EOF`).

---

## File tags & encoding — the #1 gotcha

USS on z/OS is bilingual. Every file has a **CCSID tag** (EBCDIC IBM-1047 default, or ISO8859-1 for ASCII text). AUTOCVT translates on read/write if tag matches shell locale.

### Inspect tag
```
chtag -p filename
```
Output:
- `- untagged    T=off` — no tag, no conversion
- `t IBM-1047    T=on ` — EBCDIC text
- `t ISO8859-1   T=on ` — ASCII text
- `b binary      T=off` — binary; do NOT convert

### Set tag
```
chtag -tc ISO8859-1 filename          # tag as ASCII text
chtag -tc IBM-1047 filename           # tag as EBCDIC
chtag -b filename                     # tag as binary (disable conversion)
chtag -r filename                     # remove tag (reset)
```

### Explicit conversion
```
iconv -f ISO8859-1 -t IBM-1047 ascii.sh > ebcdic.sh
iconv -f IBM-1047 -t ISO8859-1 ebcdic.txt > ascii.txt
```

### AUTOCVT
Controlled by `_BPXK_AUTOCVT` env var:
```
export _BPXK_AUTOCVT=ON        # convert per-file based on tag (default in most sites)
export _BPXK_AUTOCVT=OFF       # never convert
```

**Rule for FTP uploads**: use `binary` mode + `chtag -b` immediately + `iconv` explicitly + `chtag -tc IBM-1047`. This is the ONLY reliable pattern for uploading shell scripts. The Onda-A E2E validation proved this — see the SHC deploy runbook.

---

## Environment variables that matter

```
_BPXK_AUTOCVT=ON              # tag-based auto-conversion
_BPX_SHAREAS=YES              # child processes share TSO address space (faster)
_BPX_JOBNAME=MYJOB            # override child job name (needs BPX.JOBNAME facility)
_BPX_BATCH_UMASK=022          # umask for BPXBATCH
_CEE_RUNOPTS="FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)"    # LE runtime options
LIBPATH=/lib:/usr/lib:.       # DLL search path
STEPLIB=DB2.DSND10.SDSNLOAD   # MVS load libs visible to USS programs
PATH=/bin:/usr/bin:/rsusr/rocket/bin:/syslpp/local/acitools
LANG=C                        # or en_US.IBM-1047
_TAG_REDIR_IN=TXT             # stdin redirection tag
_TAG_REDIR_OUT=TXT            # stdout redirection tag
_TAG_REDIR_ERR=TXT            # stderr redirection tag
```

Check current: `env | grep -E '^(_BPX|_CEE|STEPLIB|LIBPATH|LANG)'`.

---

## Bridging USS and MVS

### Read/write MVS datasets from USS
```
cp "//'SYS1.PARMLIB(IEASYS00)'" /tmp/ieasys00.txt        # MVS → USS
cp -T /tmp/newmember.txt "//'C30T158.CMM.JCL(NEW)'"      # USS → PDS member (with LRECL enforcement)
```

`-T` = enforce dataset RECFM/LRECL. Without `-T`, `cp` warns on line-length mismatch.

Read a sequential dataset:
```
cat "//'C30T158.SEQ.OUT'" > /tmp/seq.txt
```

### Issue TSO commands from USS
```
tsocmd "LISTC LVL('C30T158') NAMES" 2>&1 | tee /tmp/listc.out
tsocmd "SUBMIT 'C30T158.CMM.JCL(PROBE)'"
tsocmd "STATUS PROBE(JOB12345)"
```

### Issue MVS console commands from USS
```
mvscmd "D T"                                  # simple
mvscmd -c "D IPLINFO"                         # with quoting
opercmd "D IPLINFO"                           # alternate binary
```

Both require OPERCMDS class READ (MVS.DISPLAY.*).

### Submit JCL from USS
```
submit "//'C30T158.CMM.JCL(PROBE)'"
submit < /tmp/inline.jcl                       # inline
```

Response: `JOB C30T158X(JOB12345) SUBMITTED`.

---

## BPXBATCH — run USS from JCL

The bridge from JES to USS. Run a shell script or program under a batch initiator:

```jcl
//BPXSTEP  EXEC PGM=BPXBATCH,REGION=0M,PARM='SH /u/c30t158/myscript.sh arg1'
//STDIN    DD PATH='/u/c30t158/in.txt',PATHOPTS=(ORDONLY),FILEDATA=TEXT
//STDOUT   DD PATH='/u/c30t158/out.txt',PATHOPTS=(OWRONLY,OCREAT,OTRUNC),
//            PATHMODE=(SIRWXU),FILEDATA=TEXT
//STDERR   DD PATH='/u/c30t158/err.txt',PATHOPTS=(OWRONLY,OCREAT,OTRUNC),
//            PATHMODE=(SIRWXU),FILEDATA=TEXT
```

`PARM` variants:
- `PARM='SH command args'` — run a shell command
- `PARM='PGM /path/to/binary arg1 arg2'` — exec a binary
- `PARM=''` + STDPARM DD — long parm from a dataset/file

`FILEDATA=TEXT` — enable CCSID conversion. `FILEDATA=BINARY` for binary I/O.

Alternate: **BPXBATSL** — same but calls `sh -L` (login shell, honors profile).

---

## ISHELL — ISPF-based USS browser

Enter: `ISHELL` from ISPF cmd line, or option 6 → `ISHELL`.

Panels:
- File List — Directory browse with `S/E/V/B/D/R/C/M/X/A` line commands
- File Attributes — mode, tag, owner, size, date
- Directory List — traverse tree
- Mount table — see zFS/HFS mount points

Advantages over OMVS:
- Familiar ISPF navigation
- Multi-line line commands
- No 72-char command line limit
- Safe from `FSUM2364` overlay errors

Disadvantages:
- Slower for pipe-heavy work
- No shell scripting

---

## File permissions & ACLs

```
ls -la file                    # 9-char mode string
chmod 700 file                 # rwx------ for owner only
chmod u+x,g-w file             # symbolic
chmod +t /tmp                  # sticky bit (files only removable by owner)

getfacl file                   # show ACL if present
setfacl -m u:otheruser:rx file # add ACL entry (needs FSSEC class)
```

Setuid/setgid on z/OS require `BPX.FILEATTR.PROGCTL` facility class.

---

## Filesystem mount inspection

```
df -k                          # mount table + free/used KB
df -Pv                         # verbose (POSIX + z/OS extensions)
mount                          # listing (needs MOUNT command)
```

zFS aggregates: `LISTC ENT('<hlq>.ZFS')` from TSO shows the underlying VSAM.

---

## Common pitfalls

- **`sh -n script.sh` returns RC 0 but script errors on run** — often means the shebang wasn't preserved (missing `#!/bin/sh` on line 1). Check `chtag -p` and `head -1`.
- **Script output is garbled** — untagged file being read as EBCDIC. `chtag -tc IBM-1047 script.sh` then re-source.
- **`FSUMA930 sed: Unknown option -i`** — z/OS sed lacks `-i`. Use `sed 'expr' in > out && mv out in`.
- **`ls` shows binary garbage in a text file** — file was uploaded ASCII+chtag omitted. Fix: `iconv -f ISO8859-1 -t IBM-1047 in > out; chtag -tc IBM-1047 out`.
- **`cp -T ascii "//'PDS(MBR)'"` line truncation** — target LRECL < source line length. Check LRECL with `LISTDS 'PDS'`.
- **JCL from USS has line endings LF but PDS wants EBCDIC + FB80** — `cp -T` handles this; plain `cp` does not.
- **`echo` output vanishes in `2>&1 | tee`** — z/OS shell may treat some ANSI escape sequences oddly. Test with a simple redirect first.
- **Uppercase filename gotcha** — POSIX case-sensitive; user copies from ISPF (uppercase) into USS (lowercase-preferred). `ls -la` reveals.
- **`tsocmd "OMVS"` from OMVS itself hangs** — recursion. Exit OMVS first.
