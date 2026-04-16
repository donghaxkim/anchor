# anchor

Pin your code in place. Claude can read it, but can't touch it.
AI coding assistants modify code you didn't ask them to change. They refactor nearby functions while fixing a typo, rewrite config you spent weeks tuning, and touch security-critical logic that was already audited.

`anchor` stops this. Tell Claude which code is off-limits. It can still read and reason about it, but it **physically cannot edit it**. Not an instruction it might forget. A [PreToolUse hook](https://docs.anthropic.com/en/docs/claude-code/hooks) that blocks the edit before it executes.

<img width="800" height="610" alt="anchor" src="https://github.com/user-attachments/assets/e199d576-e305-413c-8f03-bba54a29ebcc" />

## Install

```bash
git clone https://github.com/donghaxkim/anchor.git ~/.claude/skills/anchor
```

Then in your project directory, run setup once:

```bash
python3 ~/.claude/skills/anchor/scripts/anchor.py setup
```

This adds enforcement hooks to your project's `.claude/settings.json`. You only do this once per project.

**Verify it worked:** open Claude Code and run `/hooks`. You should see `Bash` and `Edit|Write|MultiEdit` hooks listed under PreToolUse.

<details>
<summary>Other install methods</summary>

**npx**
```bash
npx skills add donghaxkim/anchor
```
Then tell Claude: *"Run anchor setup for this project."*

**Plugin marketplace**
```
/plugin marketplace add donghaxkim/anchor
/plugin install anchor@donghaxkim-anchor
```
</details>

## Usage

Just talk to Claude. No commands to memorize.

```
You:     "anchor platform/services/auth_service.py - reviewed JWT auth, do not modify"
Claude:  Anchored platform/services/auth_service.py (file) - "reviewed JWT auth, do not modify"

You:     "refactor the auth system to add role-based access control"
Claude:  *tries to edit auth_service.py*

         ANCHOR VIOLATION: platform/services/auth_service.py is protected
         (reason: reviewed JWT auth, do not modify). Work around this region.

Claude:  "I see auth_service.py is anchored. I'll create a separate role_service.py
          and modify the dependencies layer instead."

You:     "check my anchors"
Claude:  All anchors OK.
```

That's it. Anchor, work, verify.

### What you can anchor

| What you say | What gets protected |
|---|---|
| "anchor src/auth.py" | Entire file |
| "protect lines 45-80 in billing.py" | Specific line range |
| "lock down the migrations folder" | Entire folder, recursively |
| "don't touch the validateToken function" | Claude finds the function boundaries, anchors those lines |

### Managing anchors

These are the commands Claude runs under the hood. You can also run them directly if you prefer:

```bash
anchor add src/auth.py --desc "security-critical"        # protect a file
anchor add src/billing.py:45-80 --desc "audited logic"   # protect specific lines
anchor add db/migrations/ --desc "production, append-only" # protect a folder
anchor list                                               # see all anchors + status
anchor check                                              # verify nothing drifted
anchor update src/auth.py                                 # accept an intentional change
anchor remove src/auth.py                                 # remove protection
```

> **Note:** `anchor` is shorthand. Claude resolves the full path automatically. To run these yourself, use the full path `python3 ~/.claude/skills/anchor/scripts/anchor.py` or add an alias:
> ```bash
> alias anchor="python3 ~/.claude/skills/anchor/scripts/anchor.py"
> ```

## How it works

When you anchor a file, `anchor.py` hashes its contents and writes an entry to a `.anchors` file at your project root. When Claude tries to edit that file, a PreToolUse hook checks the path against `.anchors`, finds a match, and blocks the edit with exit code 2. Claude receives a message explaining what's protected and why, then re-plans its approach automatically.

Three layers of enforcement:

| Layer | Catches | How |
|---|---|---|
| `enforce_edit.sh` | All Edit, Write, MultiEdit tool calls | Path matching against `.anchors` |
| `enforce_bash.sh` | `sed -i`, `echo >`, `rm`, `tee`, `cp`, `mv` | Regex extraction + path matching |
| Deny rules | Claude running `anchor remove` or `anchor update` | Claude Code permission system |

## Why not just use...

| | anchor | .cursorignore | CLAUDE.md rules | ai-guard |
|---|---|---|---|---|
| AI can still read the code | Yes | No, blocks read too | Yes | Yes |
| Enforcement | Mechanical hooks | IDE-level | None, it's a suggestion | Git hooks (post-hoc) |
| Survives context compaction | Yes | N/A | No | N/A |
| Line-range granularity | Yes | No | No | No |
| Drift detection | Yes (SHA-256) | No | No | Yes |
| AI gets context on why it's blocked | Yes | No | No | No |

The key difference: when an edit is blocked, Claude doesn't just get "permission denied." It gets told *why* that code is protected and re-plans its approach around it. No babysitting required.

## Drift detection

Anchor tracks content by SHA-256 hash. If someone modifies anchored code outside of Claude (manual edit, git merge, another tool), `anchor check` catches it:

```bash
$ anchor check
DRIFTED: src/auth.py - hash was a8f3c2e1, now 5c47d694

$ git diff src/auth.py    # review the change
$ anchor update src/auth.py    # accept it
```

The `.anchors` file is human-readable and designed to be committed to git. Your whole team shares the same protections.

```
# .anchors - managed by anchor skill
# Format: type | target | hash | description
file | src/auth.py | a8f3c2e1 | reviewed JWT auth, do not modify
lines | src/billing.py:45-80 | 7b2e1d4f | audited tax calculation
folder | db/migrations/ | c9d0e3a2 | production migrations, append-only
```

## Limitations

Anchor is a development safety tool, not a security boundary.

- **Bash enforcement is best-effort.** The hook catches common patterns (`sed -i`, `echo >`, `rm`, `tee`, `cp`, `mv`) but exotic shell one-liners could slip through.
- **Line-range anchors block the entire file in the hook.** The SKILL.md instructions tell Claude which lines to avoid. The hook is the safety net.
- **Symbol anchoring relies on Claude.** Claude reads the file, finds the function boundaries, and anchors those lines. The tool itself doesn't do AST parsing, which keeps it language-agnostic.

## Compatibility

| Platform | Enforcement |
|---|---|
| Claude Code | Full, mechanical hook enforcement |
| Codex CLI | Instruction-based via SKILL.md |
| Gemini CLI | Instruction-based via SKILL.md |
| Cursor | Instruction-based via SKILL.md |

## Technical details

- **Zero dependencies.** Python stdlib only (hashlib, argparse, json, os, sys). No pip installs.
- **No background processes.** Hooks run inline on each tool call.
- **Idempotent setup.** Running `setup` twice is safe. It merges with existing settings.
- **Git-friendly.** `.anchors` is designed to be committed and shared across a team.

## Project structure

```
anchor/
├── SKILL.md                    # Skill definition loaded by Claude
├── scripts/
│   ├── anchor.py               # Core CLI (stdlib only, zero deps)
│   ├── enforce_edit.sh          # PreToolUse hook for Edit/Write
│   └── enforce_bash.sh          # PreToolUse hook for Bash
├── references/
│   ├── hook-setup.md            # Setup documentation
│   └── anchor-format.md         # .anchors file format spec
├── assets/
│   └── settings-template.json   # Hook config template
├── marketplace.json
├── CHANGELOG.md
└── LICENSE                      # MIT
```

## License

MIT
