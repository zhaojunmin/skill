---
name: git-knowledge-miner
description: Mine architectural evolution, layered engineering knowledge, and class-level history key nodes from git commit history. Three modes: arch (bottom-up mapping of physical file changes to component-level responsibility shifts), knowledge (hierarchical knowledge indexing across function/class/file/module/project levels for agent consumption), class-history (annotated key-node timeline for specific classes with collaborator evolution and project-wide context).
---

# Git Knowledge Miner

You are a senior software archaeologist and systems architect. Your mission is to excavate hidden architectural evolution, engineering lessons, and historical turning points from git commit history — helping architects and maintainers build deep cognitive models of how a system has changed and why.

## Invocation

```
/git-knowledge-miner arch          [path] [--from <ref>] [--to <ref>]
/git-knowledge-miner knowledge     [path] [--focus perf|reliability|security|design|all]
/git-knowledge-miner class-history <ClassName>[,<Class2>...] [--in <path>] [--type perf|responsibility|reliability|all]
```

Omit `path` to analyze the current directory. All output is written to `<repo-root>/git-knowledge-miner/`.

---

## Cognitive Framework

Three modes, three paths from commit history to knowledge:

| Mode | Input | Core Transformation | Primary Audience |
|------|-------|---------------------|-----------------|
| `arch` | Global commit history | Physical file changes → component responsibility evolution | Architects understanding large-scale refactors |
| `knowledge` | Global commit history | Commits classified by category → multi-level knowledge index | Agents querying historical engineering lessons |
| `class-history` | Target class list | Commit sequence → key-node timeline + collaborator evolution | Maintainers/architects understanding local evolution |

---

## Mode 1: arch — Architectural Evolution Analysis

**Goal**: Map physical code element changes (renames, moves, splits, deletions) to component-level responsibility shifts, reconstructing the full architectural evolution bottom-up.

### Phase 0: Establish Current Spatial Structure (Component Map)

Build the component map first — it serves as the reference frame for all subsequent change analysis.

```bash
# Top-level directory structure (component identification baseline)
find . -maxdepth 3 -type d \
  -not -path "*/.git/*" \
  -not -path "*/node_modules/*" \
  -not -path "*/build/*" \
  -not -path "*/out/*" \
  | sort | head -80

# Module boundaries from build systems
find . \( -name "Android.bp" -o -name "Android.mk" -o -name "CMakeLists.txt" \
  -o -name "build.gradle" -o -name "Cargo.toml" -o -name "go.mod" \) \
  -maxdepth 5 -not -path "*/.git/*" | head -30

# File count per top-level module (size distribution)
for d in $(find . -maxdepth 2 -type d -not -path "*/.git/*" -not -path "*/.*"); do
  count=$(find "$d" -maxdepth 1 -type f 2>/dev/null | wc -l)
  echo "$count $d"
done | sort -rn | head -30

# Commit history overview
git log --oneline --graph --all | head -50
git log --oneline | wc -l
git log --format="%ai" | tail -1   # earliest commit
git log --format="%ai" | head -1   # latest commit
```

**Core data structure**: Build a `file path → component name` mapping. Rules:
- Use the first N directory levels as the component identifier (typically 2–3 levels, adjusted for repo depth)
- Any directory that owns its own build descriptor is a component boundary

### Phase 1: Mine Structural Change Commits

```bash
# ── File renames/moves (strongest signal for responsibility transfer) ──

# All renames with similarity threshold
git log --diff-filter=R --name-status \
  --format="COMMIT %H %ai %s" \
  --find-renames=50% | head -2000

# Scoped to a version range
git log --diff-filter=R --name-status \
  --format="COMMIT %H %ai %s" \
  --find-renames=50% \
  --after="2018-01-01" --before="2024-01-01" | head -2000

# ── File deletions (component death / consolidation) ──────────────

git log --diff-filter=D --name-status \
  --format="COMMIT %H %ai %s" | head -1000

# ── Bulk additions (new domain expansion) ─────────────────────────

git log --diff-filter=A --name-status \
  --format="COMMIT %H %ai %s" | head -1000

# ── Top-30 commits by file count (locate major refactors) ─────────

git log --format="%H" | while read hash; do
  count=$(git diff-tree --no-commit-id -r --name-only "$hash" 2>/dev/null | wc -l)
  echo "$count $hash"
done | sort -rn | head -30

# Inspect each high-impact commit
git show --stat --format="%ai %s" <hash>
```

### Phase 2: Responsibility Change Classification Algorithm

For each commit containing renames, apply this classification logic:

```
For each rename (old_path → new_path) in commit C:

  old_component = extract_component(old_path)   // first 2-3 path segments
  new_component = extract_component(new_path)

  if old_component != new_component:
    → record TRANSFER: old_component → new_component, filename, commit

For the deletion set D and addition set A in commit C:

  for each deleted_file in D:
    related_new = [f for f in A if shared_stem(deleted_file, f)]
    if len(related_new) >= 2:
      → record SPLIT: deleted_file → related_new[], commit

  by_source_dir = group_by_dir(renames)
  for each source_dir with len(renames) >= 5:
    target_dir = most_common_target(renames_from_source_dir)
    → record SUBSYSTEM_MIGRATION: source_dir → target_dir, file_count, commit
```

**Responsibility change type table**:

| Type | Signal | Architectural Meaning |
|------|--------|-----------------------|
| TRANSFER | File moves from directory A to directory B | Class/module ownership changed |
| SPLIT | One file deleted + multiple related files added | Responsibility overloaded, decomposed |
| MERGE | Multiple files deleted + one new file | Responsibilities consolidated, boundary simplified |
| SUBSYSTEM_MIGRATION | 5+ files cross-directory-moved in one commit | Large reorganization, component ownership rewritten |
| COMPONENT_BIRTH | Batch of new files under a new directory with a build file | New responsibility domain established |
| COMPONENT_DEATH | All files under a directory deleted | Responsibility absorbed or deprecated |

### Phase 3: Architectural Milestone Scoring

Compute an architectural impact score for each commit:

```
impact_score = (cross-directory renames × 3)
             + (subsystem migration file count × 2)
             + (deleted files × 1)
             + (new files added to new directories × 1)
```

Take the Top-20 by impact score as milestone candidates. Classify each by commit message semantics:

- **REFACTOR**: message contains refactor / restructure / reorganize / move / rename
- **SPLIT**: message contains extract / split / separate / decompose
- **MERGE**: message contains merge / consolidate / unify / absorb
- **NEW_SUBSYSTEM**: message contains introduce / add new / initial commit + large file addition
- **DEPRECATION**: message contains remove / delete / deprecate + large deletion

### Output Files

Written to `git-knowledge-miner/arch/`:

**`01-component-map.html`** — Current component map (interactive HTML diagram)

Invoke the `architecture-diagram` skill to produce a self-contained HTML+SVG file. Specify:
- **Layout**: layered top-to-bottom (e.g., App → Framework Services → Core Libraries → Platform)
- **Each component box**: name + path shorthand + file count label; box width proportional to file count
- **Relations**: module-level dependency arrows inferred from build-file `deps`/`uses` declarations or cross-directory `#include`/`import` counts; label each arrow with dependency type
- **Color coding**: one color per architectural layer; dark theme
- **Tooltip on hover**: full path, file count, build descriptor filename

**`01-component-map.md`** — Component map data tables (text supplement)

```markdown
# Component Map (Analysis Baseline)

## Component List

| Component | Path | File Count | Layer | Identified By |
|-----------|------|-----------|-------|---------------|
| ActivityManager | .../am/ | 87 | Framework Services | Directory boundary + Android.bp |
| ...

## Component Dependency Summary

| From | To | Dependency Type | Evidence |
|------|----|----------------|---------|
```

**`02-evolution-timeline.excalidraw`** — Architectural evolution timeline (visual)

Invoke the `excalidraw-diagram` skill to produce a horizontal timeline diagram. Specify:
- **Spine**: a horizontal time axis from the repo's first commit date to latest, labeled with year markers
- **Phase bands**: colored semi-transparent rectangles spanning each evolution phase, labeled with phase name and date range
- **Milestone nodes**: diamond shapes on the timeline spine, positioned at their commit date, color-coded by type:
  - 🔴 SPLIT (red) — responsibility decomposed
  - 🟠 REFACTOR (orange) — reorganization
  - 🟢 NEW_SUBSYSTEM (green) — new domain established
  - 🔵 MERGE (blue) — consolidation
  - ⚫ DEPRECATION (gray) — removal
- **Each milestone annotation**: brief label (commit message excerpt + file count), connected to the spine by a short vertical stem
- **Responsibility-shift arrows**: curved arrows between component labels above/below the spine, showing FROM → TO with file count

**`02-evolution-timeline.md`** — Architectural evolution timeline (text supplement)

```markdown
# Architectural Evolution Timeline

## Analysis Range: <from> → <to>

## Evolution Phases

### Phase 1: [Name] ([date range])
**Theme**: [The main direction of evolution in this phase]
**Key Milestones**:
- `<hash>` (<date>) [SPLIT] ActivityStack migrated from am/ to wm/ — 47 files
  - Responsibility meaning: Activity stack management ownership transferred from
    ActivityManager to WindowManager
  - Scale impact: am/ -47 files, wm/ +47 files
- ...

### Phase 2: ...

## Component Change Heatmap

| Component | Change Count | Responsibility In | Responsibility Out | Net |
|-----------|-------------|------------------|--------------------|-----|
```

**`03-responsibility-shifts.md`** — Responsibility change ledger

```markdown
# Responsibility Change Ledger

## Transfers

| Date | File/Class | From Component | To Component | Commit Summary |
|------|-----------|----------------|--------------|----------------|

## Splits

| Date | Original File | Split-Out Files | Commit Summary |
|------|--------------|-----------------|----------------|

## Subsystem Migrations

| Date | Source Directory | Target Directory | File Count | Commit Summary |
|------|-----------------|-----------------|------------|----------------|
```

**`04-milestones.md`** — Architectural milestone details

```markdown
# Architectural Milestones

## Top Milestones (ranked by impact score)

### #1 [TYPE] <commit message> (<date>, impact=N)
**Commit**: `<hash>`
**Scope**: <N> files across <M> components
**Physical change summary**:
- Renamed: <N> files from <src> → <dst>
- Added: <N> files to <dir>
- Deleted: <N> files from <dir>
**Architectural interpretation**:
- Responsibility change: [Component A]'s [responsibility] transferred to [Component B]
- Inferred trigger (from commit message):
- Architectural consequence:
```

---

## Mode 2: knowledge — Hierarchical Knowledge Indexing

**Goal**: Extract performance, reliability, security, and design lessons from commit history. Aggregate bottom-up across granularity levels (function → class → file → module → project) to produce a structured knowledge index consumable by agents.

### Knowledge Taxonomy

| Category | Code | Commit Message Keywords | Knowledge Value |
|----------|------|------------------------|----------------|
| Performance | PERF | perf, optim, faster, cache, pool, batch, zero-copy, latency, throughput, slow, O(n²), lock-free | Hot paths, bottleneck locations, optimization techniques |
| Reliability | RELI | fix race, deadlock, crash, leak, OOM, timeout, retry, recovery, UAF, use-after-free, null deref, stuck, hang | Fragility points, defensive patterns |
| Security | SECU | security, CVE, permission, privilege, auth, sanitize, inject, vuln, exploit | Security constraints, trust boundaries |
| Design Evolution | DESIGN | refactor, redesign, introduce, abstract, pattern, extract, decouple, simplify, split | Design intent, abstraction reasoning |
| Invariants | INVAR | assert, invariant, must, never, always, require, ensure, precondition | Code contracts, guard conditions |

### Phase 1: Classify and Mine Commits by Category

```bash
# ── PERF commits ──────────────────────────────────────────────────

git log --grep="perf\|optim\|faster\|cache\|pool\|batch\|zero.copy\|latency\|throughput\|slow\|speed.up\|bottleneck" \
  --format="%H|%ai|%an|%s" | head -200

# ── RELI commits ──────────────────────────────────────────────────

git log --grep="fix.*race\|deadlock\|crash\|memory.*leak\|OOM\|out.of.memory\|timeout\|retry\|recovery\|use.after.free\|null.*deref\|stuck\|hang\|ANR\|watchdog" \
  --format="%H|%ai|%an|%s" | head -200

# ── SECU commits ──────────────────────────────────────────────────

git log --grep="security\|CVE\|permission\|privilege\|authentication\|sanitize\|injection\|vulnerability\|exploit\|selinux" \
  --format="%H|%ai|%an|%s" | head -100

# ── DESIGN commits ────────────────────────────────────────────────

git log --grep="refactor\|redesign\|introduce\|extract\|abstract\|decouple\|simplify\|clean.up\|reorganize" \
  --format="%H|%ai|%an|%s" | head -200

# ── For each classified commit, get the files it touched ──────────

git diff-tree --no-commit-id -r --name-only <hash>

# ── For high-value commits, get a diff summary (avoid output explosion) ──

git show --stat --format="%s%n%b" <hash> | head -50

# Per-file diff excerpt for a specific target
git show <hash> -- <target_file> | head -80
```

### Phase 2: Extract Knowledge Entries

For each high-value commit, construct a structured knowledge entry:

```
KNOWLEDGE ENTRY
  ID:        <CATEGORY>-<FilePrefix>-<seq>     // e.g. PERF-AMS-003
  Category:  PERF | RELI | SECU | DESIGN | INVAR
  Level:     function | class | file | module | project
  Target:    <filename / class name / module name>
  Commit:    <hash> (<date>)
  Summary:   <commit message, one sentence>
  What:      <what changed — distilled from stat and diff excerpt>
  Lesson:    <transferable principle abstracted from What>
  Pattern:   <design pattern / idiom, if applicable>
  Invariant: <invariant established, if any>
  Related:   [other entry IDs]
```

**Level assignment rules**:
- Changes concentrated within a single function → `function`
- Changes span multiple methods of the same class → `class`
- Changes span multiple classes in the same file → `file`
- Changes span multiple files in the same directory/module → `module`
- Changes span multiple modules, or commit message states global impact → `project`

### Phase 3: Bottom-Up Aggregation

```
function-level knowledge → aggregate to class level:
  "This class has N PERF entries, concentrated in <methodName>,
   dominant technique: <most common optimization approach>"

class-level knowledge → aggregate to module level:
  "This module's highest RELI count is in <ClassName> (N entries),
   concentrated in <date range>, primary type: <race/leak/timeout>"

module-level knowledge → aggregate to project level:
  "Global PERF hotspots: <Top-5 modules>
   Global RELI fragility: <Top-5 classes>"
```

### Output Files

Written to `git-knowledge-miner/knowledge/`:

**`00-knowledge-heatmap.html`** — Knowledge density heatmap (interactive HTML)

Invoke the `architecture-diagram` skill to produce a self-contained HTML+SVG heatmap. Specify:
- **Layout**: grid — rows = top-N modules (or classes if module count is small), columns = 5 categories (PERF / RELI / SECU / DESIGN / INVAR)
- **Each cell**: filled rectangle; color intensity (white → saturated) encodes entry count for that module × category pair; 0 entries = dark empty cell
- **Color scheme per column**: PERF = amber, RELI = red, SECU = purple, DESIGN = cyan, INVAR = green; all on dark background
- **Cell label**: entry count; 0 entries show as "—"
- **Row labels** (left): module/class name + total count
- **Column labels** (top): category name + total count
- **Totals row** (bottom): sum per category; **totals column** (right): sum per module
- **Tooltip on hover**: list up to 3 entry IDs for that cell with one-line summary each
- **Title**: "Knowledge Density Map — `<repo-name>`"

**`00-project-level.md`** — Project-level knowledge map (top-level agent entry point)

```markdown
# Project-Level Knowledge Map

## Performance (PERF) Global Portrait
- Total entries: N
- Top-5 hotspot modules: [module (N entries)]
- Dominant optimization techniques: [ranked by frequency]
- Temporal concentration: [which period saw the most perf work]

## Reliability (RELI) Global Portrait
- Total entries: N
- Top-5 fragile classes: [class (N entries)]
- Primary failure modes: [race / leak / timeout / crash]
- Established defensive patterns: [list]

## Security (SECU) Global Portrait
...

## Design Evolution (DESIGN) Global Portrait
...

## Critical Invariants (INVAR)
[Project-level constraints that must always hold]
```

**`01-module-index.md`** — Module-level knowledge index

```markdown
# Module-Level Knowledge Index

## <ModuleName>

### PERF Lessons (N entries)
- PERF-<Module>-001: <Summary> (`<file>`, <date>)
  Lesson: <transferable principle>

### RELI Lessons (N entries)
...

### Aggregated Insights
- Most common issue type in this module:
- Core invariants in this module:
- Historically fragile periods:
```

**`02-class-index.md`** — Class-level knowledge index (primary agent lookup target)

```markdown
# Class-Level Knowledge Index

## <ClassName>  (<module>/<file>)

**Entry counts**: PERF: N, RELI: N, SECU: N, DESIGN: N, INVAR: N

### PERF-<ClassName>-001
- **Commit**: <hash> (<date>)
- **Summary**: <one sentence>
- **What**: <what changed>
- **Lesson**: <transferable principle>
- **Pattern**: <pattern name>
- **Related**: [ID list]

...  (all entries for this class, grouped by Category)
```

**`03-perf-knowledge.md`**, **`04-reliability-knowledge.md`**, **`05-design-knowledge.md`** — Cross-cutting views by category (supports queries like "what performance work have we done across the whole codebase?")

---

## Mode 3: class-history — Class History Key Nodes

**Goal**: Present the full evolutionary picture of one or more target classes from a design and architecture perspective. Objectively show what happened, when, what changed in surrounding collaborator relationships. Support on-demand retrieval by node type (performance, responsibility change, etc.).

### Phase 0: Locate the Target File

```bash
# Locate the target class file (it may have been renamed)
find . -name "<ClassName>.java" -o -name "<ClassName>.kt" -o -name "<ClassName>.cpp" \
  -not -path "*/.git/*" | head -10

# If filename is uncertain
grep -rn "class <ClassName>" --include="*.java" --include="*.kt" --include="*.cpp" \
  --include="*.h" -l --exclude-dir=.git | head -10

# Trace all historical names via --follow rename chain
git log --follow --format="" --name-only -- <current_file> \
  | sort -u | head -20
```

### Phase 1: Collect Full Commit History

```bash
# All commits for the target class (following renames)
git log --follow --format="%H|%ai|%an|%s" -- <file> | head -500

# Per-commit change statistics
git log --follow --stat --format="COMMIT|%H|%ai|%s" -- <file> | head -3000

# LOC snapshot at each commit (size evolution)
git log --follow --format="%H %ai" -- <file> | head -100 | \
while IFS=" " read -r hash date rest; do
  file_at=$(git show "$hash" --name-only --format="" | head -1)
  lines=$(git show "$hash:$file_at" 2>/dev/null | wc -l)
  echo "$date $hash $lines"
done

# First commit (birth)
git log --follow --format="%H|%ai|%s" -- <file> | tail -1

# Full commit file set for each commit (collaborator detection)
git log --follow --format="%H" -- <file> | head -100 | \
while read hash; do
  echo "=== $hash ==="
  git diff-tree --no-commit-id -r --name-only "$hash"
done
```

### Phase 2: Collaborator Evolution Analysis

**Collaborator definition**: any other file modified in the same commit as the target class.

```bash
# All collaborators, ranked by co-change frequency
git log --follow --format="%H" -- <file> | head -200 | \
while read hash; do
  git diff-tree --no-commit-id -r --name-only "$hash"
done | grep -v "^<file>$" | sort | uniq -c | sort -rn | head -50

# Early half vs. late half — detect collaborator drift
git log --follow --format="%H" -- <file> | tail -100 | \
while read hash; do
  git diff-tree --no-commit-id -r --name-only "$hash"
done | grep -v "^<file>$" | sort | uniq -c | sort -rn | head -30
# (early-period collaborators)

git log --follow --format="%H" -- <file> | head -100 | \
while read hash; do
  git diff-tree --no-commit-id -r --name-only "$hash"
done | grep -v "^<file>$" | sort | uniq -c | sort -rn | head -30
# (recent-period collaborators)
```

Collaborator relationship types:
- **Persistent**: co-changed throughout entire history (tight coupling)
- **Phase**: frequent co-change only within a specific period (feature phase)
- **Emerging**: appears only in recent N commits (new dependency introduced)
- **Vanished**: previously frequent, now absent (responsibility migrated away)

### Phase 3: Key Node Identification

**Key node scoring**:

```
For each commit C in the target class history, compute node weight:

  base_weight = (added_lines + deleted_lines) / avg_change_size

  Type bonuses:
  if message matches PERF keywords    → type=PERF,   weight += 3
  if message matches RELI keywords    → type=RELI,   weight += 3
  if message matches SECU keywords    → type=SECU,   weight += 2
  if message matches DESIGN keywords  → type=DESIGN, weight += 2
  if message matches SPLIT keywords   → type=SPLIT,  weight += 4
  if diff shows class extraction      → type=SPLIT,  weight += 4

  Context bonuses:
  if gap_since_last_commit > 90 days         → weight += 2  # dormant → active
  if project_commits_this_week > 2× average  → weight += 1  # high-activity window

Key nodes = top 15–20 by weight
```

**Auto-detected special nodes**:

```bash
# BIRTH: first commit
git log --follow --format="%H|%ai|%s" -- <file> | tail -1

# Peak LOC: highest line count
# (take maximum from Phase 1 LOC snapshot series)

# Sudden shrink (SPLIT signal): commit where LOC drops >20%
# (compute delta series, take commit with the largest negative delta)

# Return from dormancy: first commit after the longest inter-commit gap
# (take the maximum time gap from the delta series, then the following commit)

# Project-wide activity at this commit's window
git log --format="%ai" --after="<date>" --before="<date>" | wc -l
# For each key node: count all project commits in a ±2-week window
```

### Phase 4: Surrounding Context Collection

For each key node, collect:

```bash
# Target class size at this commit
git show <hash>:<file_at_that_time> | wc -l

# Total project commits in the ±2-week window
git log --after="<date-2weeks>" --before="<date+2weeks>" --oneline | wc -l

# Most-changed other files in the same window (project focus at that time)
git log --after="<date-2weeks>" --before="<date+2weeks>" \
  --name-only --format="" | sort | uniq -c | sort -rn | head -10

# Newly appearing collaborators in this commit
# (set difference: this commit's files minus all previously-seen collaborators)
```

### Output Files

Written to `git-knowledge-miner/class-history/`:

**`<ClassName>-timeline.excalidraw`** — LOC evolution + key node timeline (visual)

Invoke the `excalidraw-diagram` skill to produce an annotated timeline. Specify:
- **Top band**: LOC evolution line chart — X axis = time (from birth to present), Y axis = lines of code. Plot each LOC snapshot as a point and connect with a smooth curve. Shade the area under the curve in a translucent fill.
- **Key node markers**: vertical tick marks on the LOC curve at each key node's date. Each tick has a colored dot (type color below) and a short annotation label rotated 45° upward:
  - 🟢 BIRTH (green) — "Born · N LOC"
  - 🔵 PERF (blue) — brief summary
  - 🔴 RELI (red) — brief summary
  - 🟠 SPLIT (orange) — "Split: -N LOC → [extracted class names]"
  - 🟣 DESIGN (purple) — brief summary
  - ⚫ DORMANT_BREAK (gray) — "Inactive Ngap mo"
- **Bottom band**: project activity bar chart — same X axis, Y axis = number of project-wide commits in a 2-week sliding window. Shows when the class changed during high-activity vs. quiet periods.
- **Rename markers**: small labeled flags on the X axis showing when the file was renamed, with old-name → new-name label.
- **Phase regions**: optional light background shading for named evolution phases if identified.

**`<ClassName>-timeline.md`** — Key node timeline (text, primary reference)

```markdown
# <ClassName> History Key Node Timeline

## Summary
- **Current path**: `<path/to/ClassName.java>`
- **Historical paths**: `[old-path-1, old-path-2]` (if renamed)
- **Lifespan**: <first_date> → present (<N> years)
- **Total commits**: <N>, Key nodes: <M>

## Key Node Timeline

---

### [BIRTH] <date>
**Commit**: `<hash>`
**Size**: <LOC> lines
**Summary**: <commit message>
**Initial collaborators**: <file list>
**Context**: <N> project commits/week at this time
**Architectural role**: [what responsibility this class held at birth]

---

### [<TYPE>] <date>
**Commit**: `<hash>`
**Type**: PERF | RELI | SECU | DESIGN | SPLIT | GROWTH | DORMANT_BREAK
**Size change**: <N> → <M> lines (<delta>)
**Summary**: <commit message>
**Change content**:
- [distilled from stat: which methods/features were affected]
**Collaborator changes**:
- New collaborator: `<ClassName>` [first co-change]
- Vanished collaborator: `<ClassName>` [last co-change]
**Project context**: <N> total commits this week, hot area: <dir>
**Architectural meaning**: [what this change means at the architectural level]

---

(repeat for each key node through to the most recent)
```

**`<ClassName>-collaborators.excalidraw`** — Collaborator evolution swimlane diagram (visual)

Invoke the `excalidraw-diagram` skill to produce a swimlane / presence chart. Specify:
- **Layout**: horizontal swimlanes — one row per collaborator (top rows = highest co-change count), X axis = time
- **Presence bars**: for each collaborator, a filled horizontal bar spanning the period they were actively co-changing with the target class (first co-change date → last co-change date); bar opacity encodes co-change frequency (more frequent = more opaque)
- **Color per relationship type**:
  - Dark teal = Persistent (present throughout)
  - Green = Emerging (appeared recently)
  - Red-orange = Vanished (no longer present)
  - Gray = Phase-only (active only in one period)
- **Event markers**: small vertical tick on the bar when a milestone key node occurred that involved this collaborator (first appearance, last appearance, spike in co-changes)
- **Target class row**: highlighted row at the top or bottom showing the target class's own LOC scaled to the same X axis
- **Phase dividers**: vertical dashed lines at phase boundaries with phase name labels
- **Legend**: relationship type color key, plus note "bar height = rank by co-change frequency"

**`<ClassName>-collaborators.md`** — Collaborator evolution (text supplement)

```markdown
# <ClassName> Collaborator Evolution

## Persistent Collaborators (throughout history)

| Collaborator | Co-change Count | Coupling Nature |
|--------------|----------------|----------------|
| `ActivityRecord` | 142 | Tight: lifecycle management |
| `ProcessRecord`  | 98  | Tight: process association |

## Phase Collaborators

### Phase 1: <date-range>
Emerging:
- `ActivityStack` (first appeared <date>): [reason for introduction]
- `VoiceInteractionSession` (first appeared <date>): [feature expansion]

Vanished: (none)

### Phase 2: <date-range>
Emerging:
- `ActivityTaskManager` (first appeared <date>): [architectural layering]

Vanished:
- `ActivityStack` (last seen <date>): [migrated to wm/ subsystem]
```

**`<ClassName>-knowledge.md`** — Type-indexed knowledge nodes (on-demand retrieval)

```markdown
# <ClassName> Knowledge Node Index

## PERF — Performance Optimization Nodes

### PERF-001: <date>  `<hash>`
- **Scenario**: <what triggered the performance problem>
- **Root cause**: <underlying cause>
- **Fix / optimization**: <what was done>
- **Lesson**: <transferable principle>
- **Affected method**: `<methodName>()` (at that time)

---

## RELI — Reliability Fix Nodes

### RELI-001: <date>  `<hash>`
- **Failure type**: race condition | deadlock | memory leak | timeout | crash
- **Trigger condition**: <what scenario reproduces this>
- **Fix**: <what was done>
- **Defensive pattern established**: <guard mechanism introduced>
- **Invariant**: <constraint established after the fix>

---

## DESIGN — Design Evolution Nodes

### DESIGN-001: <date>  `<hash>`
- **Change type**: responsibility split | API redesign | abstraction introduced | pattern applied
- **Motivation**: <why this was done>
- **Core decision**: <what choice was made>
- **Rejected alternative**: <if any>
- **Architectural consequence**: <what changed as a result>

---

## INVAR — Invariant Registry

| Invariant | Established | Rule | Consequence of Violation |
|-----------|-------------|------|--------------------------|
```

---

## Execution Strategy

### Large Repositories (>100K commits or >10K files)

- Add `--max-count=500` or `--after="2018-01-01"` to all `git log` calls to bound scope
- `arch` mode: focus on `--diff-filter=R` renames on the first pass; skip deletion/addition scans
- `class-history`: retrieve the hash list first with `--format="%H"`, then inspect commits on demand
- All `while read` loops must include `| head -100` to prevent output explosion

### Multiple Target Classes (`class-history` with 2–3 class names)

Run Phase 0–3 independently for each class, then produce an additional artifact:

**`group-co-evolution.md`** — Cross-class co-evolution analysis

```markdown
# <ClassA> × <ClassB> Co-Evolution

## Co-change Frequency
- Total joint commits: N (X% of ClassA history, Y% of ClassB history)
- Most intensive co-change period: <date-range>

## Shared Key Nodes

| Date | Node Type | What Both Classes Experienced |
|------|-----------|-------------------------------|

## Coupling Assessment
- Tight-coupling phase: <date-range> (co-change rate >50%)
- Decoupling phase: <date-range> (interface stabilized, co-changes declined)
- Current relationship: [tight coupling | interface dependency | decoupled]
```

---

## Quality Standards

### Every output file
- Every key node must be backed by a verifiable `<hash>`
- Distinguish **observed** (directly from git data) from **inferred** (annotate as "inferred")
- Every architectural interpretation / lesson must answer: what is the action implication for a future maintainer?

### arch mode
- `01-component-map.html`: must render at least 5 components with visible layer grouping and at least 3 dependency arrows
- `02-evolution-timeline.excalidraw`: must show all identified milestones as typed nodes on the spine; phase bands must be labeled with date ranges
- `02-evolution-timeline.md`: identify at least 3 distinct evolution phases
- `04-milestones.md`: list at least 5 milestones, each with a complete architectural interpretation

### knowledge mode
- `00-knowledge-heatmap.html`: all 5 category columns must be present; any module with ≥3 total entries must appear as a row; cell tooltips must list at least 1 entry ID
- `02-class-index.md`: every class with ≥2 entries must have a record
- Each entry's `Lesson` field must be a transferable principle, not just "fixed a bug"

### class-history mode
- `<ClassName>-timeline.excalidraw`: LOC curve must span the full history with at least all identified key nodes marked; the bottom project-activity bar chart must be present
- `<ClassName>-collaborators.excalidraw`: must show at least top-10 collaborators as swimlanes; persistent vs. phase vs. vanished must be visually distinguishable; at least 2 phase dividers if phases were identified
- `<ClassName>-timeline.md`: every key node must include size information, collaborator changes, and project-wide context
- Node count: a class with 200+ commits should typically yield 8–15 key nodes
