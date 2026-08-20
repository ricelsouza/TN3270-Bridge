# ISPF Editor & View Reference

ISPF Edit (option 2) and View (option 1) are the workhorses for reading/writing datasets. This reference focuses on non-destructive patterns; MUTATION commands are flagged.

---

## Enter Edit or View

From ISPF Primary Menu:

```
mainframe_send fieldName=OPTION text="2" aid=ENTER
# fill Dsname Level or full DSN
mainframe_send labelLeft="Project" text="C30T158" aid=NONE
mainframe_send labelLeft="Group" text="CMM" aid=NONE
mainframe_send labelLeft="Type" text="JCL" aid=NONE
mainframe_send labelLeft="Member" text="PROBE" aid=ENTER
```

Or via 3.4:
```
mainframe_send fieldName=OPTION text="3.4" aid=ENTER
mainframe_send labelLeft="Dsname Level" text="C30T158.CMM.JCL" aid=ENTER
# on member list, position cursor on member row, col 1, type E for Edit or V for View
```

Or use `mainframe_read_dataset` (safer, read-only, restores prior panel):
```
mainframe_read_dataset dsn=C30T158.CMM.JCL member=PROBE
```

---

## Edit profile — set once per session

The command line (`===>`) accepts:

| Cmd | Effect |
|---|---|
| `CAPS OFF` | Preserve mixed case (default is CAPS ON which upper-cases input) |
| `NUM OFF` | Hide sequence numbers cols 73-80 |
| `NUM ON STD` | Show standard sequence numbers |
| `NUM ON COBOL` | COBOL sequence numbers (cols 1-6) |
| `NULLS ON` | Trailing nulls when saving (prevents blank-space rewrite) |
| `NULLS OFF` | Trailing spaces preserved |
| `AUTOSAVE OFF` | Do not save on PF3 without prompt |
| `AUTOSAVE ON PROMPT` | Prompt on PF3 (safer default) |
| `AUTONUM OFF` | Do not renumber on save |
| `AUTOLIST OFF` | Do not create list on save |
| `PROFILE` | Show current profile settings |
| `PROFILE <name>` | Switch profile (e.g. `PROFILE COBOL`, `PROFILE JCL`) |
| `PROFILE LOCK` | Freeze current profile |
| `PROFILE RESET` | Reset to system defaults |
| `RECOVERY ON` | Enable recovery table (space-hungry) |
| `RECOVERY OFF` | Disable |
| `TABS OFF` | Disable tab expansion |
| `HEX ON VERT` | Show HEX zones vertically |
| `HEX OFF` | Hide HEX |
| `HILITE ASM/COBOL/JCL/PLI/REXX/AUTO` | Syntax highlight |
| `COLS` | Overlay column ruler |
| `BNDS` | Show current bounds |
| `BOUNDS n m` | Set editable bounds (cols n through m) |
| `RESET` | Clear all pending line commands + labels + finds |

Recommended default for JCL/REXX authoring:
```
CAPS OFF
NUM OFF
NULLS ON
AUTOSAVE ON PROMPT
HILITE JCL     (or REXX / COBOL)
```

---

## Primary commands (line `===>`)

### Navigation
| Cmd | Effect |
|---|---|
| `TOP` / `T` | Top of data |
| `BOTTOM` / `BOT` / `B` | Bottom of data |
| `LOCATE nnnn` | Go to line nnnn (with `NUM ON`) |
| `LOCATE .LABEL` | Go to named label |
| `LOCATE FIRST` / `LAST` / `NEXT` / `PREV` `LABEL/CHANGE/COMMAND/ERROR/EXCLUDED/SPECIAL` | Filter navigation |

### Find / Change
| Cmd | Effect |
|---|---|
| `FIND 'text'` / `F 'text'` | Case-insensitive substring find |
| `F 'text' ALL` | Highlight all occurrences |
| `F P'.'` | Find picture pattern (`.` = any char) |
| `F X'C1'` | Find hex byte |
| `F 'text' PREFIX` / `SUFFIX` / `WORD` | Boundary-aware find |
| `F 'text' NEXT` / `PREV` / `FIRST` / `LAST` / `ALL` | Direction |
| `RFIND` (PF5) | Repeat last find |
| `CHANGE 'old' 'new'` / `C 'old' 'new'` | Change first occurrence |
| `C 'old' 'new' ALL` | Change all — **MUTATION** |
| `RCHANGE` (PF6) | Repeat last change |
| `EXCLUDE 'text' ALL` / `X 'text' ALL` | Hide lines containing text |
| `RESET EXCLUDED` | Show all again |

### Blocks
| Cmd | Effect |
|---|---|
| `CUT` (line cmd `C/CC`) | Cut to clipboard `.DEFAULT` |
| `CUT .LABEL` | Cut to named clipboard |
| `PASTE` (line cmd `A/B`) | Paste after (A) or before (B) |
| `COPY dsn(mbr)` (line cmd `A/B`) | Copy dataset content |
| `MOVE dsn(mbr)` | Move (deletes source) — **MUTATION** on source |
| `CREATE dsn(mbr)` (line cmds `C/CC`) | Create new member from block — **MUTATION** |
| `REPLACE dsn(mbr)` | Overwrite member — **MUTATION** |
| `SUBMIT` | Submit current buffer as JCL (may or may not save first) |

### Utilities
| Cmd | Effect |
|---|---|
| `SORT` | Sort lines (with column range) — **MUTATION** in Edit |
| `COMPARE dsn(mbr)` | Show diff vs another member |
| `COMPARE *` | Compare against previous saved state |
| `HEX ON VERT / OFF` | Toggle hex display |
| `RECOVERY ON` | Enable UNDO |
| `UNDO` | Undo last change (needs RECOVERY ON) |
| `REDO` | Redo |
| `MODEL <type>` | Insert JCL/REXX/CLIST/COBOL/PL/I skeleton |
| `SAVE` | Save without exit |
| `CANCEL` | Exit without save |
| `END` (PF3) | Save + exit (or prompt) |

---

## Line commands (leftmost 6 cols)

| Cmd | Effect | MUTATION? |
|---|---|---|
| `I` | Insert 1 line below | Yes |
| `I5` | Insert 5 lines below | Yes |
| `D` | Delete line | Yes |
| `D5` / `DD..DD` | Delete 5 lines / block | Yes |
| `R` / `R5` | Repeat line 1 / 5 times | Yes |
| `RR..RR` | Repeat block | Yes |
| `C` / `CC..CC` | Copy line/block to clipboard | No (until paste) |
| `M` / `MM..MM` | Move line/block | Yes on source |
| `A` | Paste after this line | Yes |
| `B` | Paste before this line | Yes |
| `O` | Overlay from clipboard | Yes |
| `X` / `X5` / `XX..XX` | Exclude from display | No (view-only) |
| `S` / `S5` | Show excluded (opposite of X) | No |
| `F` / `L` | First/last excluded shown | No |
| `COLS` | Show column ruler at this line | No |
| `BNDS` | Show bounds line here | No |
| `TABS` | Show tabs line | No |
| `MASK` | Show mask line | No |
| `NOTE` | Insert note (does not save to member) | No |
| `TE` | Text entry mode (word wrap) | Yes |
| `TS` | Text split (break line here) | Yes |
| `TF` | Text flow (reformat paragraph) | Yes |

Numeric suffix (`I5`) or double letters (`CC..CC` marking begin/end) work on all.

---

## Member management (PDS)

From member list panel (3.4 → dataset → member list):

| Cmd on member | Effect | MUTATION? |
|---|---|---|
| `S` / `E` | Edit | Potentially |
| `V` | View | No |
| `B` | Browse | No |
| `D` | Delete member | Yes |
| `R newname` | Rename | Yes |
| `C newmbr` | Copy | Yes on target |
| `M newmbr` | Move | Yes |
| `X` | Reset user data (statistics) | Yes on stats |
| `I` | Show ISPF stats (user, ver.mod, size, date) | No |

Primary commands from member list:
| Cmd | Effect |
|---|---|
| `SORT NAME` / `SIZE` / `MOD` / `USER` / `DATE` / `TIME` | Sort |
| `FILTER pattern` | Filter member names (e.g. `FILTER *TEST*`) |
| `RESET` | Clear filter |
| `SUBMIT membername` | Submit as JCL |

---

## View vs Edit vs Browse

| Mode | Editable | Save | Line cmds | Session | Use when |
|---|---|---|---|---|---|
| **View** | Yes | Yes (asks for target DSN) | Yes | Recoverable | Read-then-edit-if-needed |
| **Edit** | Yes | Yes (same DSN) | Yes | Recoverable | Author/modify a member |
| **Browse** | No | N/A | No | None | Read-only; no accidental changes |

`mainframe_read_dataset` uses View internally + restores prior panel — the safest programmatic path.

---

## Anti-patterns

- **Editing SYS1.PARMLIB directly** — SYSCTLG-level datasets. Use a private copy.
- **Forgetting `CAPS OFF`** on REXX/JCL that requires mixed case (JSON/YAML strings, USS paths).
- **`SAVE` on a locked profile** — ISPF may reject with `PROFILE LOCKED`.
- **`RECOVERY OFF` + long edit** — no UNDO. Turn ON before big change sessions.
- **`NUM ON` on non-numbered file** — cols 73-80 get overwritten with sequence numbers if you save.
- **Editing a member two people opened** — last save wins. Use `SDSF ENQ` to check enqueue holder before editing shared members.
