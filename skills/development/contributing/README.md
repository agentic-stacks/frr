# Contributing to FRR

## Fork and Branch Workflow

### Initial Setup

```bash
# 1. Fork FRRouting/frr on GitHub

# 2. Clone your fork
git clone https://github.com/<you>/frr.git
cd frr

# 3. Add upstream remote
git remote add upstream https://github.com/FRRouting/frr.git

# 4. Keep master up to date
git fetch upstream
git checkout master
git merge upstream/master
```

### Feature Branch Workflow

```bash
# Create a feature branch from latest master
git checkout master
git pull upstream master
git checkout -b fix/bgp-update-parsing

# ... develop, commit ...

# Push to your fork
git push -u origin fix/bgp-update-parsing

# Open a PR on GitHub targeting FRRouting/frr master
```

### Branch Naming Conventions

| Branch type | Target |
|---|---|
| `master` | New features and non-critical bug fixes |
| `dev/MAJOR.MINOR` | Pre-release branch, created 4 weeks before release |
| `stable/MAJOR.MINOR` | Long-term support, renamed from dev/ at release |

Always target `master` for new contributions unless you are backporting a fix.

## Coding Style

### C Code

FRR follows **Linux kernel coding style** with these specifics:

- Format with `clang-format` using the project's `.clang-format` file
- Alternative: `tools/indent.py` for FRR-specific macros (DEFUN, etc.)
- Run `checkpatch.sh` before submission

```bash
# Format a file
clang-format -i bgpd/bgp_route.c

# Format FRR-specific macro blocks
python3 tools/indent.py bgpd/bgp_route.c

# Check a patch
git diff HEAD~1 | ./tools/checkpatch.sh -
```

### Key Style Rules

| Rule | Detail |
|---|---|
| Indentation | Tabs (8-wide) for C code |
| Line length | 80 characters preferred, 100 maximum |
| Braces | Opening brace on same line (except functions) |
| Pointer style | `char *p` not `char* p` |
| Function declarations | `void foo(void)` not `void foo()` |
| Comments | C-style `/* */` not C++ `//` |

### Prohibited Functions

| Prohibited | Replacement |
|---|---|
| `strcpy()` | `strlcpy(dst, src, sizeof(dst))` |
| `strcat()` | `strlcat(dst, src, sizeof(dst))` |
| `sprintf()` | `snprintf(buf, sizeof(buf), ...)` |
| `system()` | Do not use; avoid fork/exec patterns |
| `memset(x, 0, ...)` | Use initializer `= { 0 }` or `= {}` |

### Required Practices

- Use C99 fixed-width types: `uint8_t`, `uint32_t` (not `u_int8_t`, `u_char`)
- Use typesafe containers from `lib/typesafe.h` (not legacy `linklist.c`)
- Use `DEFPY` for all new CLI commands (not `DEFUN`)
- Use `sizeof(*ptr)` in allocations (not `sizeof(struct foo)`)
- Use YANG/northbound for new configuration where the daemon supports it

### Python Code

```bash
# Format all Python code with black
black --check tests/topotests/my_test/
black tests/topotests/my_test/
```

### YANG Models

```bash
# Validate -- output should be identical to input (minus comments)
yanglint -f yang yang/frr-ripd.yang
```

## Commit Message Format

### Structure

```
<subsystem>: <imperative summary, max 72 chars>

<body -- wrapped at 72 chars, explain WHY not WHAT>

Signed-off-by: Your Name <your@email.com>
```

### Rules

| Rule | Detail |
|---|---|
| Title prefix | Subsystem name + colon: `bgpd:`, `lib:`, `zebra:`, `doc:`, `tests:` |
| Multiple subsystems | Comma-separated: `lib,bgpd: clean up clang warnings` |
| Imperative mood | "Fix", "Add", "Remove" -- not "Fixed", "Added", "Removed" |
| Max title length | 72 characters |
| No trailing period | `bgpd: fix route selection` not `bgpd: fix route selection.` |
| Body separator | Blank line between title and body |
| Body wrapping | 72 characters (except sample output) |
| Markdown allowed | Code blocks with triple backticks, bullet lists with `- ` |
| End-user perspective | Title should make sense in a release changelog |

### Examples

```
bgpd: fix crash when receiving malformed OPEN message

A malformed OPEN message with zero hold timer caused a null pointer
dereference in bgp_open_receive(). Add validation before accessing
the timer structure.

Signed-off-by: Jane Developer <jane@example.com>
```

```
lib,bgpd: convert to typesafe route hash

Replace the legacy hash implementation with DECLARE_HASH from
typesafe.h. This eliminates unchecked casts and improves type safety.

Signed-off-by: John Developer <john@example.com>
```

## Signed-off-by and DCO

Every commit must include a `Signed-off-by` line. This certifies the Developer Certificate of Origin (DCO):

1. You created the contribution (wholly or in part) and have the right to submit it, OR
2. It is based on prior work with a compatible open-source license, OR
3. It was provided to you by someone who certified (1) or (2)

```bash
# Add Signed-off-by automatically
git commit -s -m "bgpd: fix route leak on session reset"

# Amend a commit to add Signed-off-by
git commit --amend -s
```

## Pull Request Process

### Before Submitting

Run the pre-submission checklist:

```bash
# 1. Format code
clang-format -i <changed-files>
python3 tools/indent.py <changed-files>

# 2. Run checkpatch
git diff master... | ./tools/checkpatch.sh -

# 3. Build successfully
make -j$(nproc)

# 4. Run unit tests
make check

# 5. Build documentation (if doc changes)
make html

# 6. Build distribution tarball
make dist
```

### Creating the PR

| Field | Guidance |
|---|---|
| Title | High-level technical summary (matches commit title for single-commit PRs) |
| Description | Context for reviewers; for single-commit PRs, copy the commit message |
| Target branch | `master` for new work; `stable/X.Y` for backports |

### Squash Requirements

Before merge, consolidate these into logical commits:

- Fix/review feedback commits
- Typos and formatting corrections
- Merge/rebase commits
- Work-in-progress snapshots

Ensure the final commit messages capture everything -- PR descriptions are not preserved in git history.

### After Submission

- CI runs automatically; expect results within ~2 hours
- **Fix failing CI before expecting review** -- the community ignores PRs with failing tests
- Respond to all reviewer comments
- Use "Re-request review" after addressing feedback
- Never delete or dismiss someone else's review comments

### PR Lifecycle

- PRs older than 180 days are closed automatically
- Closed PRs can be reopened or recreated easily
- Trivial PRs may merge immediately when CI passes
- Non-trivial PRs receive time for community review interest

## Code Review

### Standards

- At least one review from someone other than the author is required before merge
- Anyone may review a patch
- Review depth should match patch scope and impact

### GitHub Review Marks

| Mark | Meaning |
|---|---|
| Approve (green checkmark) | Reviewer is willing to merge |
| Changes Requested (red X) | Rejection by maintainer -- must be resolved |
| Comment (yellow circle) | Needs time but not a merge blocker |
| Comment (text bubble) | Should be read but not a merge blocker |

### Conflict of Interest

Submissions from individuals at one company should be merged by someone unaffiliated with that company.

## CI Checks

Every PR runs through automated CI:

| Check | Requirement |
|---|---|
| Compilation | Must build on all supported platforms |
| Unit tests | `make check` must pass |
| Topotests | Full topotest suite must pass |
| Style | `checkpatch.sh` and `clang-format` compliance |
| Documentation | Builds without warnings |
| Packages | Distribution packages build correctly |

PRs with failing CI or unresolved "Changes Requested" reviews cannot be merged.

## Documentation Requirements

### Code Documentation

| Scope | Requirement |
|---|---|
| Header functions | Descriptive comment above signature (return value, params, summary) |
| Static functions | Comment if behavior is not immediately obvious |
| Global variables | Comment on usage |
| New code in `lib/` | All above are **hard requirements** |

### User Documentation

For features with user-visible functionality, add docs in `doc/user/`:

```rst
.. clicmd:: show ip bgp neighbor A.B.C.D detail

   Display detailed information about the specified BGP neighbor.

.. code-block:: frr

   router bgp 65001
    neighbor 10.0.0.1 remote-as 65002
```

### Documentation Build

```bash
# HTML docs
make html

# PDF docs (requires LaTeX)
make latexpdf

# Both must build with zero warnings
```

### Formatting Rules

- 3-space indents under RST directives
- Cross-references: lowercase alphanumeric + hyphens only
- 80-character line wrapping where possible

## License

FRR is GPLv2-or-later. All new files require the SPDX header:

```c
// SPDX-License-Identifier: GPL-2.0-or-later
/*
 * Description of file
 * Copyright (C) 2026  Your Name
 */

#include <zebra.h>
```

Submitted code must be GPLv2-or-later or a compatible license (e.g., MIT) that permits GPLv2 redistribution.

## Release Process

### Schedule

FRR releases follow a 4-month cycle on the **first Tuesday of March, July, and November**.

Version scheme: `MAJOR.MINOR.BUGFIX`

### Timeline

| Milestone | Timing | Action |
|---|---|---|
| Feature freeze | 6 weeks before release | No new features on master |
| Branch creation | 4 weeks before release | `dev/X.Y` branch created; master unfrozen |
| RC tag | 2 weeks before release | `frr-X.Y.Z-rc` tagged on release branch |
| Release | Release date | Branch renamed `stable/X.Y`; release tagged |

### Backporting

- Bug fixes: backported to the two most recent stable releases
- Security fixes: backported to all releases <= 1 year old
- Every newer stable branch must also receive the fix
- Each backport requires justification and release manager approval

## Mailing Lists

| List | Purpose |
|---|---|
| `dev@lists.frrouting.org` | Development discussion |
| `frog@lists.frrouting.org` | Users and operators |
| `announce@lists.frrouting.org` | Release announcements |
| `security@lists.frrouting.org` | Security reports (private) |
| `tsc@lists.frrouting.org` | Technical Steering Committee (private) |

## Reverting Changes

When reverting a full PR, revert each commit individually rather than reverting the merge commit:

```bash
# Preferred -- preserves granularity for git bisect
git revert <commit-sha-1>
git revert <commit-sha-2>

# Avoid -- merges remain in history, complicates future merges
git revert -m 1 <merge-commit-sha>
```
