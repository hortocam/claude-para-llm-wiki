# claude-para-llm-wiki — Project Plan

## Vision

Build a distributable Claude Code plugin that gives anyone a fully-operational,
PARA-structured Obsidian vault with an LLM-maintained wiki, hardened inbox ingestion
workflows, and enforced tool routing. The plugin is opinionated by default but
configurable — install once, run forever.

**Design principles:**
- Agent burns tokens to reason, Python scripts handle mechanical transforms
- Defense in depth: structural rules + runtime scanning, not either/or
- PARA method as the organizational spine (not raw/wiki/dev)
- Files are the truth — no database, no external service required to operate

---

## Plugin Architecture

This repository (`claude-para-llm-wiki`) **is** the plugin. Users install it once,
then use `vault-init` to scaffold any number of vaults.

### Repository layout

```
claude-para-llm-wiki/
├── package.json              # npm package descriptor (npx-runnable)
├── install.sh                # Platform-aware installer
├── vault-CLAUDE.md           # Template CLAUDE.md written to each vault
├── routing-rules.md          # Default PARA routing rules for para-ingest
│
├── skills/
│   ├── obsidian-markdown/    # Adapted from kepano/obsidian-skills
│   ├── obsidian-bases/       # Adapted from kepano/obsidian-skills
│   ├── json-canvas/          # Adapted from kepano/obsidian-skills
│   ├── obsidian-cli/         # Adapted from kepano/obsidian-skills
│   ├── defuddle/             # Adapted from kepano/obsidian-skills
│   ├── qmd/                  # QMD search skill (integrated from existing)
│   ├── vault-init/           # Vault scaffolding skill
│   ├── para-ingest/          # PARA inbox routing skill
│   ├── wiki-ingest/          # PARA-aware wiki ingestion skill
│   ├── wiki-query/           # QMD-first vault query skill
│   ├── adr-writing/          # ADR skill (01-Projects/{name}/ADRs)
│   └── incident-writing/     # Incident skill (01-Projects/{name}/Incidents)
│
├── commands/
│   ├── vault-init.md
│   ├── para-ingest.md
│   ├── wiki-ingest.md
│   └── wiki-query.md
│
├── hooks/
│   ├── pre-tool-use.sh       # Tool routing + injection scan gate
│   ├── post-tool-use.sh      # Hot cache update + agent log
│   └── hook-settings.json    # Hook event bindings
│
├── src/                      # Python utilities
│   ├── injection_scan.py     # LLM Guard PromptInjectionScanner
│   ├── html_to_md.py         # HTML → Markdown (trafilatura)
│   ├── transcribe.py         # mlx-whisper (Apple Silicon) / faster-whisper
│   ├── hot_cache.py          # Builds .hot-cache.json from 90-Wiki/
│   ├── router.py             # PARA routing rules engine
│   └── requirements.txt
│
├── templates/
│   ├── daily/daily-note.md
│   ├── projects/project-index.md
│   ├── projects/adr.md
│   ├── projects/incident.md
│   ├── areas/area-index.md
│   ├── resources/resource-note.md
│   ├── wiki/concept.md
│   ├── wiki/entity.md
│   ├── wiki/synthesis.md
│   └── shared/meeting-note.md
│
└── config/
    └── obsidian/             # .obsidian config to copy at vault-init
        ├── app.json
        ├── core-plugins.json
        ├── community-plugins.json
        └── plugins/
            ├── dataview/
            ├── templater-obsidian/
            ├── obsidian-tasks-plugin/
            └── daily-note-navbar/
```

---

## Vault Structure

Every vault scaffolded by `vault-init` follows this layout:

```
{vault-root}/
├── 00-Inbox/                     # Drop folder — all ingestion starts here
├── 01-Projects/
│   └── {project-name}/
│       ├── _index.md
│       ├── ADRs/
│       └── Incidents/
├── 02-Areas/                     # Ongoing areas of responsibility
├── 03-Resources/                 # Reference material
├── 04-Archives/                  # Completed / inactive
├── 05-Daily/
│   └── {YYYY}/
│       └── {MM}/
├── 90-Wiki/                      # LLM-maintained wiki
│   ├── concepts/
│   ├── entities/
│   ├── syntheses/
│   ├── agent-log/                # Daily agent activity log
│   └── index.md
├── _Templates/
│   ├── Daily/
│   ├── Projects/
│   ├── Areas/
│   ├── Resources/
│   └── Wiki/
├── .claude/
│   ├── skills/                   # Installed from plugin
│   └── commands/                 # Installed from plugin
├── CLAUDE.md                     # Vault operational rules
└── .hot-cache.json               # QMD hot cache
```

**Numeric prefixes** (`00-`, `01-`, ..., `90-`) keep folders sorted lexically
in filesystem views and terminal output. `_Templates` sorts before numerics,
`90-Wiki` sorts after active work folders.

---

## Zone Rules

The vault has four zones with distinct agent access rules, encoded in CLAUDE.md:

| Zone | Folders | Agent access | Notes |
|------|---------|--------------|-------|
| Inbox | 00-Inbox/ | Write (drop only) | para-ingest clears it |
| Active | 01-Projects/, 02-Areas/ | Collaborative write | Agent proposes, you approve major edits |
| Reference | 03-Resources/, 04-Archives/ | Read + append | Agent never reorganizes |
| Wiki | 90-Wiki/ | Agent owns | Agent creates/edits freely; you rarely touch |

**05-Daily/** is yours — agent reads to synthesize, never edits.
**_Templates/** is yours — agent reads, never modifies.

---

## Key Workflows

### vault-init
1. Prompt user for vault root path and vault name
2. Create PARA folder structure (00-Inbox through 90-Wiki + _Templates)
3. Copy CLAUDE.md from vault-CLAUDE.md template, substituting vault name
4. Copy skills/ and commands/ into vault .claude/
5. Copy hook scripts + update .claude/settings.json with hook bindings
6. Copy templates/ into vault _Templates/
7. Copy config/obsidian/ into vault .obsidian/ (preserving existing if present)
8. Write routing-rules.md to vault root
9. Run hot_cache.py to initialize empty .hot-cache.json
10. Report scaffold summary

### para-ingest (runs before wiki-ingest in any session)
1. Scan 00-Inbox/ for new files
2. For each item, detect content type:
   - Audio (mp3/m4a/wav/ogg) → transcribe.py → outputs new .md in 00-Inbox, re-scan
   - HTML file → html_to_md.py → replace with .md version
   - URL in .txt → html_to_md.py (fetch + extract) → .md
3. Run injection_scan.py on each extracted text
4. Quarantine flagged items (move to 00-Inbox/.quarantine/ with warning note)
5. For clean items, call router.py with filename + content summary
6. router.py consults routing-rules.md, returns destination path
7. Present routing plan to user for approval
8. After approval, move files to destinations + update 90-Wiki/agent-log/

### wiki-ingest
1. Accept URL or file path as argument
2. If URL: run html_to_md.py, then injection_scan.py on result
3. If file: run injection_scan.py on content
4. If scan flags content: halt, report to user
5. Identify 3-7 concepts and 1-3 entities
6. Check 90-Wiki/concepts/ and 90-Wiki/entities/ for existing pages
7. Present ingestion plan (new/updated pages, wikilinks to add) — wait for approval
8. Execute plan after approval
9. Update 90-Wiki/index.md if genuinely new concept added
10. Trigger hot_cache.py update (via post-tool hook or explicit call)
11. Write action to 90-Wiki/agent-log/{today}.md

### wiki-query
1. Accept natural language question
2. Check .hot-cache.json first — tokenize query, match against indexed frontmatter + summaries
3. If cache hits found: read matched files, synthesize answer
4. If cache miss: fall back to grep-based search across 90-Wiki/
5. Answer in prose, cite all consulted files via [[wikilinks]]
6. Signal clearly when inferring vs. quoting vault content
7. Suggest `/wiki-ingest <source>` when vault lacks coverage on the topic

---

## Python Utilities (src/)

All Python utilities are invoked by hooks or commands — never by the agent reasoning
over the task. This keeps token costs low and execution deterministic.

### injection_scan.py
- Library: `llm-guard` (ProtectAI)
- Scanner: `PromptInjectionScanner` with configurable threshold (default 0.75)
- Input: file path or stdin text
- Output: JSON `{safe: bool, score: float, excerpt: str}`
- Flagged content: moved to `00-Inbox/.quarantine/` with `QUARANTINE.md` warning note
- Cross-platform: pure Python, no GPU required

### html_to_md.py
- Primary: `trafilatura` (best-in-class content extraction)
- Fallback: `BeautifulSoup4` + custom cleanup for sites trafilatura misses
- Preserves: title, author, date, source URL (written to frontmatter)
- Strips: ads, nav, cookie banners, duplicate content
- Output: `.md` file with YAML frontmatter

### transcribe.py
- Platform detection: `sys.platform` + check for `torch.backends.mps.is_available()`
- Apple Silicon Mac: `mlx_whisper` (on-device, fastest)
- Universal fallback: `faster_whisper` with `compute_type="auto"` (CPU/CUDA)
- Model size: configurable via `--model` flag (default `base`, `medium` for accuracy)
- Output: `.md` file in 00-Inbox/ with frontmatter (source-file, duration, word-count, transcript-date)
- Audio in, markdown out — no intermediate formats stored

### hot_cache.py
- Scans all `.md` files in `90-Wiki/`
- Extracts: frontmatter (title, type, tags, sources), first 200 chars of body
- Writes: `.hot-cache.json` at vault root — array of `{path, title, type, tags, summary}`
- Incremental mode: `--update <changed-file>` patches a single entry without full rebuild
- Full rebuild: `--rebuild` or called by vault-init
- Used by wiki-query for near-zero-token search

### router.py
- Reads `routing-rules.md` from vault root (agent-editable routing config)
- Parses markdown table into rule objects: `{pattern, destination, priority}`
- Matches filename + content summary against rules (regex + keyword)
- Returns ranked destination path suggestions
- Ambiguous items: returns top 2 with confidence scores for agent to present to user

---

## Defense Strategy

### Layer 1 — Runtime content scanning (LLM Guard)
Every piece of external content (fetched URL, uploaded file, audio transcript)
passes through `injection_scan.py` before the agent sees it. Flagged content
is quarantined, never passed to context.

### Layer 2 — Structural defenses (CLAUDE.md + allowed-tools)
- CLAUDE.md zone rules: agent cannot write to Daily or Templates
- Slash command `allowed-tools` whitelists: wiki-ingest cannot `rm`, para-ingest
  cannot write to 90-Wiki/ directly
- Plan-before-execute gates: wiki-ingest and para-ingest both show plans and
  await explicit approval before any file moves

### Layer 3 — Audit trail
Every agent file operation is appended to `90-Wiki/agent-log/{YYYY-MM-DD}.md`
by the post-tool hook. Provides human-readable audit of what the agent did and when.

### Layer 4 — Hook tool routing enforcement
Pre-tool hook intercepts:
- `grep`/`find` calls against vault content → injects QMD correction message
- Direct file reads on vault content → suggests Obsidian skills/MCP alternative

---

## Transcription Strategy

```
Audio file dropped in 00-Inbox/
         │
         ▼
  Platform detection
    (transcribe.py)
         │
    ┌────┴────┐
    │         │
Apple         Other
Silicon       (Mac Intel,
              Linux, Win)
    │         │
mlx_whisper  faster_whisper
(Metal/MPS)  (CPU/CUDA auto)
    │         │
    └────┬────┘
         │
  transcript.md → 00-Inbox/
  (frontmatter: source, duration,
   word-count, transcript-date)
         │
  para-ingest re-scans → routes
```

Model selection guidance (written to vault CLAUDE.md for user reference):
- `base`: Fast, good enough for clear speech, meeting notes
- `small`: Better accuracy, still fast
- `medium`: Best quality, slower — use for important recordings
- `large-v3`: Maximum quality — only if you have Apple Silicon or GPU

---

## Cross-Project Vault Integration

Any code project can link to the vault. On `vault-init`, two variables are
established in the project's local CLAUDE.md or `.env.claude`:

```
VAULT_BASE_PATH=/Users/cameron/vault-personal
VAULT_PROJECT_PATH=01-Projects/my-project-name
```

All skills, commands, and hooks read these variables to resolve paths. This means:
- ADRs created in a code project session land in the correct vault project folder
- Incidents, meeting notes, and code context flow to the right PARA location
- A `vault-link` command (future) will let you register a code repo to a vault project

The skills are vault-location agnostic — the same plugin works for personal vault
on one machine and work vault on another.

---

## Obsidian Plugin Compatibility

The plugin ships pre-configured for these already-installed plugins:

| Plugin | Integration |
|--------|-------------|
| `dataview` | Templater and `.base` files use dataview queries for project/task views |
| `homepage` | vault-init sets homepage to 90-Wiki/index.md |
| `templater-obsidian` | All templates use Templater syntax for dynamic dates/paths |
| `obsidian-tasks-plugin` | Meeting note and project templates include tasks blocks |
| `obsidian-excalidraw-plugin` | json-canvas skill produces compatible canvas format |
| `iconic` | Folder icons configured in app.json for visual PARA hierarchy |
| `auto-todo-toggle` | Task templates use compatible checkbox syntax |
| `daily-note-navbar` | Daily note path configured for 05-Daily/{YYYY}/{MM}/ |

---

## Distribution

The plugin is distributed as an npm package:

```bash
# Install to a vault (interactive)
npx claude-para-llm-wiki@latest init

# Or: clone and run installer directly
git clone https://github.com/cameronhorton/claude-para-llm-wiki
./install.sh /path/to/vault
```

`install.sh` is idempotent — safe to run multiple times (updates skills/commands
without overwriting vault content or user-edited CLAUDE.md).

---

## Backlog (v2+)

| ID   | Feature | Notes |
|------|---------|-------|
| BL-1 | vault-init for existing vault | Detect + merge mode; no overwrite of existing content |
| BL-2 | PDF handler in para-ingest | pdfminer.six / pymupdf → markdown |
| BL-3 | Video transcription | yt-dlp extract audio → transcribe.py |
| BL-4 | RSS / feed auto-ingest | Cron-triggered para-ingest from feed list |
| BL-5 | Path 2 MCP integration | Optional obsidian-local-rest-api support for Dataview queries |
| BL-6 | Cross-vault sync | Personal ↔ work shared concept references |

---

## Team

| Role | Agent | Scope |
|------|-------|-------|
| Project Manager | agent-organizer | Coordinates phases, reviews plans, unblocks |
| Skill Developer | ai-engineer | skills/, commands/, vault-CLAUDE.md, config/ |
| Python Developer | python-pro | src/ utilities (all 5 scripts) |
| Reviewer | code-reviewer | Review gate after each phase |
