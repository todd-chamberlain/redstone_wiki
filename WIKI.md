# Redstone Wiki — Schema & Conventions

Code-indexed knowledge map for the **redstone** GPU cluster preflight validation suite, inspired by [Karpathy's LLM Knowledge Base pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). A persistent, cross-referenced markdown knowledge base designed for Obsidian. The wiki lives upstream of the redstone codebase; redstone is pulled in as a git submodule at `redstone/`.

## Getting Started

### 1. Clone and init the submodule

```bash
git clone <this-repo-url> redstone-wiki
cd redstone-wiki
git submodule update --init
```

If the redstone repo lives at a different URL than what `.gitmodules` specifies, override it:

```bash
git submodule set-url redstone <your-redstone-repo-url>
git submodule update --init
```

### 2. Open in Obsidian

Read this wiki in [Obsidian](https://obsidian.md):

1. Open Obsidian
2. **Open folder as vault** → select the `redstone-wiki/` directory
3. Start at `index.md` — it links to every page

The `redstone/` submodule includes its own docs (`redstone/docs/`) with Mermaid diagrams and threshold justifications. These appear in the Obsidian graph view alongside the wiki pages — the [Architecture Overview](architecture/overview.md) links to them so you can navigate between the wiki's component walkthroughs and the codebase's compact reference docs.

All navigation stays inside Obsidian. Component pages have the source code embedded inline.

### 3. Browsing the raw source

The `redstone/` submodule contains the full codebase. You can open any file directly for the complete source beyond the wiki excerpts. The wiki pages are self-contained: every module is explained from the component pages alone.

## Repo Structure

```
redstone-wiki/
├── redstone/               # Git submodule → redstone codebase
├── WIKI.md                 # This file — schema and conventions
├── index.md                # Master index linking all pages
├── log.md                  # Append-only changelog
├── results.md              # Results overview + conclusion
├── results/                # Per-layer result breakdowns (11 pages)
├── results-data/           # Raw JSON result files
├── components/             # One page per test script (9 pages)
├── architecture/           # System-level understanding (4 pages)
├── infrastructure/         # Provisioning, build, deploy (4 pages)
├── patterns/               # Design decisions + rationale (5 pages)
└── reference/              # Quick-lookup tables (3 pages)
```

## How Component Pages Work

Each component page embeds the key source code inline with the design rationale:

- Code in fenced `python` blocks alongside the decisions behind it
- Cross-references between wiki pages (all `.md`, all in-vault)
- No links to `.py` files — the code lives in the page

## Conventions

1. **Cross-references use relative markdown links** — `[orchestrator](../components/orchestrator.md)`
2. **No external file links** — all code is shown inline in component pages
3. **Tables use GitHub-flavored markdown**
4. **No orphan pages** — every page appears in `index.md`
5. **Append-only changelog** — every wiki edit gets a `log.md` entry

## Updating When Redstone Changes

When the redstone codebase is updated:

```bash
# Pull the latest redstone commit into the submodule
git submodule update --remote

# Check what changed
cd redstone && git log --oneline -5 && cd ..
```

Then re-index:

1. **Check code-index.md** — verify line numbers in the symbol table still match. If functions moved, update the line numbers.
2. **Check component pages** — if a function's logic changed, update the inline code excerpts and narrative. Only update what changed.
3. **Check thresholds.md** — if new thresholds were added or defaults changed, update the reference table.
4. **Add a log.md entry** describing what changed.
5. **Commit the submodule pointer update** along with any wiki page changes.

The submodule pins to a specific commit, so wiki references stay valid until you explicitly update. There's no risk of the wiki drifting from the code it documents.
