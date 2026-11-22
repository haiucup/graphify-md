# Graphify Knowledge Vault (Obsidian)

A setup guide for turning any repository into a knowledge graph with graphify, rendered as an Obsidian vault in `docs/graphify/`. It is written so you can copy this one file into a new repo and hand it to Claude Code, Gemini CLI, Codex, or another coding agent. The graph, notes, and maintenance workflow are shared; only the agent integration differs.

**How to use this file**
1. Copy it into the new repo, for example as `docs/GRAPHIFY.md`.
2. Open your coding agent in that repo and send the one-shot init prompt:
   > Read docs/GRAPHIFY.md and run the full init for the current agent. Detect the agent host and its available tools; do not assume Claude Code. First run the local switch check (§5.0), reuse a valid saved answer, and stop after setting it if the answer is N:
   > (1) `.graphifyignore` for this stack (§3);
   > (2) the tooling from the Appendix in `graphify-out/` plus `graphify-out/.graphify_python` (§2), with `curate.py` CONFIG adapted (§4.1–4.2);
   > (3) `.gitignore` (§5.1);
   > (4) shared rules in `docs/GRAPHIFY_RULES.md` plus loaders in root `CLAUDE.md`, `GEMINI.md`, and `AGENTS.md` (§5.2), preserving existing instructions;
   > (5) Graphify integration for the current or explicitly selected agent platforms (§2.1) and the comment policy with linter config (§5.3).
   > Use the CLI fallback (§2.2) when skills or slash commands are unavailable. If delegation is unavailable, process note batches sequentially (§4.6). Do not install unrelated agent platforms or configure paid model providers automatically. Finish with the verification checklist in §5.2.
   > Then follow the path in §4.0 that matches this repo: no code (stop), fresh framework skeleton (baseline, §4.8) or existing project code (Level 2, §4.3–4.7). If you are unsure which applies, ask me.
3. In an existing repo with a lot of code, run it in two steps instead:
   > Read docs/GRAPHIFY.md and set up Level 1 and Level 3. Stop and show me the result before Level 2.

   > Continue with Level 2 from docs/GRAPHIFY.md.

From then on the comment policy keeps the code clean: explanations go into the vault after every task, not into code comments (§5.3).

---

## 1. What you get

```
source code + docs in the repo
      │  (1) agent skill or Graphify CLI (§2.2)        → nodes + relations
      ▼
graphify-out/graph.json  (raw: many duplicates, one node per method)
      │  (2) curate.py           → fold methods into classes, merge duplicates (curation_map.json)
      │  (3) install_curated.py  → clustering, community names, GRAPH_REPORT.md
      ▼
graphify-out/graph.json  (curated)
      │  (4) graphify export obsidian --dir docs/graphify   → one .md per node + Connections
      │  (5) enrich_vault.py     → inserts the written note from node_notes.json
      ▼
docs/graphify/*.md  → open as a vault in Obsidian
```

| Level | Result | Cost |
|---|---|---|
| 1. Raw vault | One note per node, links only, many duplicates | Minutes, cheap |
| 2. Curated vault | Deduplicated nodes, one full written note per node (Overview / Details / Why / Gotchas / Where) | ≈ 7k tokens per node for notes |
| 3. Agent integration | Agents query the graph and read notes without being told | Shared rules + agent-specific instruction loaders (§5.2) |

**Key idea:** in Level 2 the "brain" of the vault is `graphify-out/node_notes.json`. The `.md` files in `docs/graphify/` are rendered output. Never edit them by hand, because the next export overwrites them.

### Files

| Path | Role | Commit? |
|---|---|---|
| `graphify-out/graph.json` | Current (curated) graph | No, regenerable |
| `graphify-out/graph.pre-curation.json` | Raw graph, input of `curate.py` | No |
| `graphify-out/curate.py` | Rule-based folding + duplicate merging | Yes |
| `graphify-out/install_curated.py` | Clustering, community names, report | Yes |
| `graphify-out/enrich_vault.py` | Inserts written notes into the vault | Yes |
| `graphify-out/graphify_tools.py` | Helpers: missing notes, batches, merging notes | Yes |
| `graphify-out/curation_map.json` | Duplicate mapping written by an LLM review | Yes |
| `graphify-out/community_names.json` | Human names for communities | Yes |
| `graphify-out/node_notes.json` | One written note per node id | Yes |
| `graphify-out/notes_work/` | Batches, note-writing prompt, temp files | No |
| `graphify-out/GRAPH_REPORT.md`, `graph.html` | Overview: god nodes, communities | No |
| `docs/graphify/` | Obsidian vault | Yes |
| `graphify-out/notes.lang` | Language of the written notes, `en` or `id` (§5.0, "Notes language") | Yes, shared by every clone |
| `.graphify.local` | Local switch for the Obsidian vault (§5.0) | No, local to each clone |
| `docs/GRAPHIFY_RULES.md` | Shared runtime instructions for all agents (§5.2) | Yes |
| `CLAUDE.md`, `GEMINI.md`, `AGENTS.md` | Agent entrypoints that load the shared rules (§5.2) | Yes; preserve existing rules |

---

## 2. Prerequisites

- Python 3.10+, [uv](https://docs.astral.sh/uv/), a coding agent with repository read/write and shell access, and Obsidian to view the exported vault. Claude Code is one supported host, not a requirement.
- Install the shared CLI once per machine:
  ```bash
  uv tool install graphifyy==0.9.66
  ```
  Pin the version: `enrich_vault.py` uses internal helpers of `graphify.export`. Re-test the pipeline before upgrading. Check the package version with the installed interpreter's `importlib.metadata.version("graphifyy")`; do not assume `graphify --version` is supported.
- `graphify-out/.graphify_python` holds the interpreter that has graphify installed. A skill may write it, but CLI-only init must create it when missing:
  ```bash
  mkdir -p graphify-out
  printf '%s\n' "$(uv tool dir)/graphifyy/bin/python" > graphify-out/.graphify_python
  ```
  This is a POSIX-shell example. On Windows, use the actual interpreter path reported by the installation and equivalent PowerShell commands. Validate that the saved interpreter can import `graphify` before running the pipeline.

### 2.1 Choose the agent integration

Choose by **agent host**, not by model name: a host controls instruction files, skills, tools, and hooks; a model alone does not. Changing a model within the same host does not require a separate graph or vault.

Run `graphify install --help` to verify supported platform names and flags in the installed version. From the repository root, choose the matching command below; `--project` keeps the integration in this repository. Do not run every row automatically.

| Agent host | Project integration example | Root instruction entrypoint |
|---|---|---|
| Claude Code | `graphify install --project --platform claude` | `CLAUDE.md` |
| Gemini CLI | `graphify install --project --platform gemini` | `GEMINI.md` |
| Codex | `graphify install --project --platform codex` | `AGENTS.md` |
| Other hosts supported by the installed CLI | Use their exact `--platform` value from `graphify install --help`; check scope support | The host's documented instruction/rules file |
| Generic Agent Skills host | `graphify install --project --platform agents`, if the host reads `.agents/skills/` | `AGENTS.md` only if the host actually reads it |
| Host without skill integration | No skill installation required; use §2.2 | Add a loader to its supported instruction file, or explicitly attach/read the shared rules at session start |

For a user-wide skill installation, omit `--project` only when that scope is intended. Install integrations only for the current host and any additional hosts the user selected. The three small root loaders in §5.2 can still be prepared for portability without installing all three hosts' skills or hooks.

Inspect installer output to locate the actual `SKILL.md` and any generated instruction sections or hooks. Account for custom profile paths; do not assume every host uses `~/.claude/skills/`. Read the matching skill when available. Preserve existing project instructions and hook configuration when merging. A skill installation alone does not guarantee that the shared project rules are loaded: complete §5.2 too.

### 2.2 Portable execution and fallback

Use the host's installed Graphify skill when it is exposed. `/graphify .` and `/graphify . --update` are **agent commands**, not shell commands; do not assume every host supports that spelling. Read the installed skill to find the host's invocation.

The CLI works independently of slash commands:

| Need | Portable CLI path | Scope |
|---|---|---|
| Build or refresh the code graph | `graphify update .` | Code extraction; no semantic document extraction |
| Inspect existing knowledge | `graphify query "<question>"`, `graphify explain "<Node>"`, `graphify path "<A>" "<B>"` | Uses the existing graph |
| Full extraction with a configured provider | `graphify extract . --backend <supported-backend>` | Code + semantic extraction; requires the selected provider's configuration |
| Render the vault | `graphify export obsidian --dir docs/graphify` | Follow with `enrich_vault.py` as described below |

Check `graphify --help` for supported extraction backends and flags. For example, the pinned CLI supports `gemini`, `claude`, `openai`, `kimi`, `deepseek`, and `ollama`; those backend names are separate from agent platform names such as `codex`. An authenticated agent session does not by itself configure the CLI's provider credentials. Do not print secrets or automatically choose a paid provider.

If no suitable skill or configured extraction provider is available, use `graphify update .` for a code-only graph and report that documents have not been semantically indexed. Do not claim a full code-and-docs init. If the host cannot run a shell, prepare the commands for the user and report which steps remain unexecuted. Missing slash commands or delegation do not by themselves block CLI-based initialization.

---

## 3. Level 1: raw vault

1. Create `.graphifyignore` in the repo root. Always exclude graphify's own output, or the graph re-ingests itself:
   ```text
   # graphify output (vault + build artifacts) - never re-ingest
   docs/graphify/
   graphify-out/
   # dependencies / build output (adjust per stack)
   vendor/
   node_modules/
   dist/
   build/
   target/
   storage/
   .venv/
   ```
2. Build the graph using the current host's skill or the CLI path in §2.2. Use the skill/full extraction for code + docs when configured; otherwise run:
   ```bash
   graphify update .
   ```
   Report this fallback as code-only. Do not enter `/graphify .` into a shell.
3. Export the vault:
   ```bash
   graphify export obsidian --dir docs/graphify
   graphify export html                     # optional: graphify-out/graph.html
   ```
4. In Obsidian: *Open folder as vault*, choose `docs/graphify`, then open Graph view. Notes carry `community/<name>` tags, so you can colour the graph by community.

**Prompt for the agent:**
> Set up Level 1 from docs/GRAPHIFY.md: create `.graphifyignore` for this stack, build the graph using the current host's skill or the CLI fallback (§2.2), state whether documents were indexed, export the vault to `docs/graphify`, and report node/edge/community counts plus the top god nodes from `GRAPH_REPORT.md`.

---

## 4. Level 2: curated vault with written notes

### 4.0 Which path

| Repo state | Path |
|---|---|
| No code yet | Stop after init. The first post-task routine (§5.3) builds the graph, and every node in it is your own code, so it gets a note. |
| Fresh framework skeleton (`laravel new`, `nest new`, Spring Initializr, `create-next-app`, …) with nothing of your own yet | **Baseline** (§4.8): build the graph and mark the scaffolding as "no note needed". Notes then grow per task. |
| Existing code you wrote | **Full Level 2** (§4.3–4.7): curate, map duplicates, write notes for everything once. |

How to tell a skeleton apart: the git history holds only the initial commit or scaffold, and the files match what the framework's installer generates. If in doubt, ask.

### 4.1 Install the tooling

Create the four scripts from the **Appendix** inside `graphify-out/`:
- `curate.py` (A)
- `install_curated.py` (B)
- `enrich_vault.py` (C)
- `graphify_tools.py` (D)

Then create `graphify-out/notes_work/PROMPT.md` from Appendix E.

### 4.2 Adjust `curate.py` to the stack

Edit only the `CONFIG` block at the top:

| Setting | What to set |
|---|---|
| `EXTERNAL_RX` | Prefixes of framework/vendor symbols to drop (e.g. `Illuminate`, `org.springframework`, `@nestjs`) |
| `CODE_ROOTS` | Folders that hold code (`app/`, `src/`, `tests/`, …) |
| `CODE_EXTS` | Source extensions whose file node is merged into the class it is named after: `Name.<ext>` (PHP, Java), `kebab-case.role.<ext>` (`app-parameter.service.ts` → `AppParameterService`, `user.entity.ts` → `User`) and `<timestamp>-Name.<ext>` migrations. A file without such a class (utils, constants, specs) keeps its own node and note |
| `SMALL_SYMBOL_RX`, `TEST_FILE_RX` | Symbols folded into their file node instead of becoming nodes: UPPER_CASE constants and enum members, framework page exports (Next.js `metadata`, fonts), and everything inside test files. Their notes become a "Folded symbols" list in the file's note |
| `MANIFESTS` | Dependency manifests whose keys fold into the manifest node |
| `DOC_PRIORITY` | Doc paths, most authoritative first; decides which duplicate survives |
| `HUBS`, `USED_BY` | Optional: group or link files that would otherwise be orphans |

**Prompt:**
> Adapt the CONFIG block of `graphify-out/curate.py` to this repo. Check which framework symbols appear as code nodes in `graphify-out/graph.json` and which folders hold code. Do not change anything below CONFIG.

### 4.3 Phase 1: rule-based folding

```bash
PY=$(cat graphify-out/.graphify_python)
cp graphify-out/graph.json graphify-out/graph.pre-curation.json   # keep the raw graph
$PY graphify-out/curate.py --phase1-only
```
Check the printed stats. Methods folded into classes and dropped framework symbols should account for most of the reduction.

### 4.4 Phase 2: duplicate map (LLM review, once)

```bash
$PY graphify-out/graphify_tools.py doc-nodes     # → notes_work/doc_nodes.json
```

**Prompt:**
> Build `graphify-out/curation_map.json`. Inputs:
> - `graphify-out/notes_work/doc_nodes.json`: every non-code node of the raw graph, with label, source and neighbours.
> - `graphify-out/graph.curated.json`: the phase-1 output, which holds the code node ids.
>
> For each doc-derived node that duplicates another node, add an entry. Typical duplicates are the same concept under different names, a doc mention of a code class, or an endpoint described in several docs. Use one of two forms:
> - `{"action": "merge", "key": "<id>", "label": "<canonical label>"}`: the nodes are the same thing. `key` is either an existing code node id, or a new stable kebab-case id for a concept, such as `jwt-authentication`. Every node in a group uses the same key.
> - `{"action": "fold", "into": "<id>"}`: the node is a minor detail of another node and becomes one of its members.
>
> Leave unique nodes out. The code is the source of truth. Work in chunks of about 150 nodes and merge each chunk into the file, so progress survives an interruption. Validate the JSON at the end.

Then:
```bash
$PY graphify-out/curate.py && $PY graphify-out/install_curated.py
```

### 4.5 Community names

`install_curated.py` prints each community under the label of its most-connected node.

**Prompt:**
> Write `graphify-out/community_names.json` as `{"<label printed by install_curated.py>": "<2–5 word community name>"}`. Base the names on what the members of each community have in common, then re-run `install_curated.py`.

### 4.6 Write the notes

1. Edit `notes_work/PROMPT.md`: fill in the project description, the "verified facts" list and `<LANGUAGE>` from `graphify-out/notes.lang` (§5.0, "Notes language"). These facts stop every batch agent from re-deriving or contradicting the same things.
2. Create the batches:
   ```bash
   $PY graphify-out/graphify_tools.py batches --size 28   # → notes_work/nbatch_NN.json (nodes without a note)
   ```
3. If the host supports delegation and it is permitted, use one subagent per batch, at most 4 at a time. Otherwise process the same batches sequentially in the current agent. Keep the same output files and checkpointing; do not require a model switch or another agent product.

**Prompt:**
> For every `graphify-out/notes_work/nbatch_NN.json`, use one subagent when supported and permitted, at most 4 at a time; otherwise handle each batch sequentially yourself. Give each one this task: "Follow `graphify-out/notes_work/PROMPT.md`. Your batch: `notes_work/nbatch_NN.json`. Output: `notes_work/notes_NN.json`. Temp folder: `notes_work/tmp_NN/`." When all batches are done, run `graphify_tools.py merge-notes` and then `graphify_tools.py missing`. Re-batch any leftovers. Collect the doc drift and bugs the agents report into `notes_work/FINDINGS.md`.

> ⚠️ Cost: about 7k tokens per node. Start with the core communities if the graph is large: edit the batch files or delete the ones you want to skip.

### 4.7 Render the vault

```bash
$PY graphify-out/graphify_tools.py where-lines        # refresh line numbers in each note's Where
graphify export obsidian --dir docs/graphify
$PY graphify-out/enrich_vault.py docs/graphify
graphify export html
```
Each note now has the written note and the curation facts (Sources, Also known as, Members, Framework dependencies) between `<!-- graphify-notes:start -->` and `<!-- graphify-notes:end -->`, followed by the Connections list.

### 4.8 Fresh framework skeleton: baseline instead of notes

Framework scaffolding is already documented by the framework, so writing notes for it wastes tokens. Build the graph once, without the LLM, and mark every existing node with an empty note:

```bash
PY=$(cat graphify-out/.graphify_python)
graphify update .                                              # code only, no LLM
cp graphify-out/graph.json graphify-out/graph.pre-curation.json
$PY graphify-out/curate.py --phase1-only && $PY graphify-out/install_curated.py
$PY graphify-out/graphify_tools.py baseline                    # scaffolding → empty note ("no note needed")
graphify export obsidian --dir docs/graphify && $PY graphify-out/enrich_vault.py docs/graphify
```

After that:
- `missing` lists only nodes added by your own work, so the post-task routine (§5.3) writes notes only for those.
- When a task changes a scaffolding node (for example a new relation on the default `User` model), its empty note is replaced with a real one.
- Skip §4.4 and §4.5 for now. Run the duplicate map once docs (PRD, specs) have grown, and name communities once features have taken shape.

---

## 5. Level 3: agent integration

### 5.0 Local Obsidian switch

Each clone decides for itself whether it uses the Obsidian vault. The choice lives in `.graphify.local` in the repo root, holds one line, and is ignored by git because it is local:

```text
graphify-obsidian=Y
```

**Check it** at the start of any graphify work:

```bash
case "$(sed -n 's/^graphify-obsidian=//p' .graphify.local 2>/dev/null)" in
  Y) echo "on" ;;
  N) echo "off" ;;
  *) echo "ask" ;;
esac
```

| Result | What to do |
|---|---|
| `ask` | The file is missing or its value is not valid. Ask the user: "Will you use graphify Obsidian in this repo? (Y/N)". Save the answer, then continue with the result below. |
| `on` | Check that graphify and Obsidian are installed, and help install whatever is missing (see below). Then use the whole pipeline: written notes, vault rendering, and the post-task routine (§4.6, §4.7, §5.3, §7). |
| `off` | Do not write `node_notes.json`, do not run `graphify export obsidian` or `enrich_vault.py`, and skip the post-task routine. `graphify query`, `explain` and `path` still work on the last built graph. |

**When the result is `on`, check the tools are installed on this machine:**

```bash
if command -v graphify >/dev/null 2>&1; then echo "graphify CLI: ok"; else echo "graphify CLI: missing"; fi
# Check the selected host's SKILL.md at the path reported by its installer (§2.1).
# A host without skill support uses the CLI fallback (§2.2).
if command -v obsidian >/dev/null 2>&1 \
  || flatpak info md.obsidian.Obsidian >/dev/null 2>&1 \
  || snap list obsidian >/dev/null 2>&1 \
  || [ -n "$(find "$HOME" -maxdepth 5 -iname 'Obsidian*.AppImage' -not -path '*/.cache/*' -print -quit 2>/dev/null)" ] \
  || [ -d /Applications/Obsidian.app ]; then echo "obsidian: ok"; else echo "obsidian: missing"; fi
```

Check the package version with the interpreter from §2. If required tooling is missing or graphify is not version 0.9.66, help install it before continuing. A missing host skill is not a CLI failure: use §2.2 when appropriate. Obsidian is the viewer; if it cannot be installed here, report that limitation separately from graph extraction/export.

| Tool | Install |
|---|---|
| graphify | `uv tool install graphifyy==0.9.66`, then install only the selected host integration (§2.1), or use the CLI fallback (§2.2). If `uv` is missing, install it first from [docs.astral.sh/uv](https://docs.astral.sh/uv/). |
| Obsidian on Linux | `flatpak install -y flathub md.obsidian.Obsidian`, or `sudo snap install obsidian --classic` |
| Obsidian on macOS | `brew install --cask obsidian` |
| Obsidian on Windows | `winget install Obsidian.Obsidian` |

If an install needs `sudo` or a password, ask the user to run that command themselves, then run the check again. On Windows, confirm Obsidian from the Start menu or `winget list Obsidian.Obsidian`, because the check above is for Linux and macOS.

**Save the answer:**

```bash
echo "graphify-obsidian=Y" > .graphify.local   # or graphify-obsidian=N
```

Change your mind later by running the same command with the other value.

**Notes language.** The language of the written notes is a repo decision, not a per-clone one, because every clone shares `node_notes.json`. It lives in `graphify-out/notes.lang`, which is committed and holds one line: `en` (English) or `id` (Indonesian).

**Check it** before writing or updating any note:

```bash
lang=$(tr -d '[:space:]' < graphify-out/notes.lang 2>/dev/null)
case "$lang" in en|id) echo "$lang" ;; *) echo "ask" ;; esac
```

| Result | What to do |
|---|---|
| `ask` | Ask the user: "Write the graphify notes in English or Indonesian? (en/id)". Save the answer with `echo en > graphify-out/notes.lang` (or `id`), set `<LANGUAGE>` in `notes_work/PROMPT.md` to match, then continue. |
| `en` / `id` | Write the body of every new or updated note in that language. |

In both languages these stay as they are: the H2 headings (`## Overview`, `## Details`, `## Why`, `## Gotchas & current state`, `## Where`), because `enrich_vault.py` and `where-lines` look for them; bold node names, code identifiers, paths and commands; and UI strings, quoted verbatim. Changing the language later applies to new and updated notes only. Existing notes are not translated unless the user asks, because translating costs tokens for every note.

### 5.1 `.gitignore`

```text
# graphify: keep only the curated notes and tooling; everything else is regenerable
/graphify-out/*
!/graphify-out/node_notes.json
!/graphify-out/curation_map.json
!/graphify-out/community_names.json
!/graphify-out/curate.py
!/graphify-out/install_curated.py
!/graphify-out/enrich_vault.py
!/graphify-out/graphify_tools.py
!/graphify-out/notes.lang
# local choice of each clone: graphify-obsidian=Y or N
/.graphify.local
```

### 5.2 Shared rules and agent entrypoints

During init, create `docs/GRAPHIFY_RULES.md` from the shared snippet below. This is the runtime source of truth for Graphify across agents; `docs/GRAPHIFY.md` remains the portable setup guide. In an existing repository, merge the rules and preserve project-specific decisions instead of overwriting them.

Create or merge a small loader in **all three** root entrypoints: `CLAUDE.md`, `GEMINI.md`, and `AGENTS.md`. For other agents, add the same loader to their supported instruction file when identified. Keep engineering, UI/UX, OpenSpec, nested-directory, and generated framework rules intact. If Graphify rules already exist in several entrypoints, consolidate only those sections into the shared file and leave a loader in each; do not replace entire files or create references that point back at one another.

Use plain instructions, so the loader does not depend on one host's `@file` import syntax:

```markdown
## Shared Graphify instructions

Before analyzing or changing this repository, read and follow
`docs/GRAPHIFY_RULES.md`. It defines the local switch, graph lookup,
notes, comment policy, and post-task refresh for every coding agent.
Also follow more specific instructions in the directory you are working in.
```

Use the same repository-relative path in every entrypoint. Merely creating `AGENTS.md` does not guarantee an unknown host reads it: configure that host's documented instruction mechanism, or explicitly supply the shared rules at session start.

**Shared snippet for `docs/GRAPHIFY_RULES.md`:**

```markdown
## Knowledge graph (graphify)
- Before any graphify work, run the local switch check from docs/GRAPHIFY.md §5.0. On `ask`, ask the user whether this
  repo uses graphify Obsidian and save the answer to `.graphify.local`. On `on`, follow every rule here. On `off`,
  skip notes, vault rendering and the post-task refresh.
- The code is the source of truth. When docs or graph notes disagree with it, follow the code and flag the drift.
- For architecture or "where/how does X work" questions, check the graph first:
  `graphify query "<question>"`, `graphify explain "<Node>"`, `graphify path "<A>" "<B>"`.
  Then read the node's note in docs/graphify/, especially "Gotchas & current state".
- Before editing a class, component, or module, read its related docs/graphify notes when available.
  Check the actual source before trusting a note; report missing or stale graph context.
- Never edit docs/graphify/*.md by hand. Notes live in graphify-out/node_notes.json and are rendered by
  graphify-out/enrich_vault.py.
- Temp files for note writing go in graphify-out/notes_work/tmp_NN/, never in /tmp.
- Other agents may work in this repo at the same time. Read a shared file (graphify-out/node_notes.json,
  community_names.json, curation_map.json, docs) right before you change it, change only the entries your task
  touches, and write it back at once; never write back a copy read earlier in the session. Delete a note only when
  a `graphify update` + curation run in the same step shows its node is gone.
- Write note bodies in the language in graphify-out/notes.lang (`en` or `id`). When the file is missing, ask the
  user "English or Indonesian?" and save the answer there (docs/GRAPHIFY.md §5.0, "Notes language"). Headings
  stay in English.

## Comments: context lives in the vault, not in the code
- Do not write explanatory comments. No class or method summaries, no step-by-step comments, no commented-out
  code, and no docblock tags that only repeat a type hint.
- Allowed: type annotations that tools need (generics, magic properties, checked exceptions), and a one-line
  "why" comment where code is intentionally unusual and someone would otherwise "fix" it.
- Put explanations, design rationale, business rules and gotchas in the node's note in
  graphify-out/node_notes.json (sections Overview / Details / Why / Gotchas & current state / Where).
- An empty note ("") marks framework scaffolding that intentionally has no note. When a task changes such a
  node, replace the empty string with a real note.
- At the end of every task that changes code, run the code-only refresh from docs/GRAPHIFY.md §5.3:
  update + curate + install, write notes for new nodes, update notes of changed nodes, render the vault.
  Run full semantic extraction of code + docs only when asked, using the host skill or CLI (§2.2).
  If a command fails or cannot run, report the incomplete step; do not claim the graph was refreshed.
```

**Init verification checklist:**

- `docs/GRAPHIFY_RULES.md` exists and each root entrypoint (`CLAUDE.md`, `GEMINI.md`, `AGENTS.md`) explicitly points to it.
- Existing project instructions are preserved, and loaders do not form a circular chain.
- The current host has actually read the shared rules; an unsupported host has an explicit loading mechanism.
- `.graphify.local` and `graphify-out/notes.lang` are reused when valid; the same clone and vault are shared across hosts.
- The CLI and saved Python interpreter work; the chosen host's skill/hook setup is checked if installed. Report any unavailable viewer separately.
- The chosen graph-building path has succeeded, or unexecuted steps and code-only/document-indexing limits are listed. For the no-code path (§4.0), explicitly report that graph creation is deferred.
- Delegation is optional; a host without it can finish the same note batches sequentially.
- Report which entrypoints and integrations were prepared, the active switch/language, extraction scope, and any remaining limitations. Do not claim another agent was tested merely because its loader was created.

### 5.3 Comment policy

The goal is to keep source files free of AI-written prose: the "why" goes into the graph note of the node it belongs to, and the code stays short. This is set up once, during init.

**What stays in code and what goes to the vault**

| In code | In `node_notes.json` |
|---|---|
| Type annotations that tools read: generics (`array<string, int>`, `List<User>`), ORM magic properties (`@property` on models), `@throws` on public methods | What a class or method does (Overview / Details) |
| A one-line "why" on code that is intentionally unusual (a disabled check, a workaround, a magic number) | Design rationale and rejected alternatives (Why) |
| Framework-required annotations and attributes | Business rules, flows, error codes, config it reads |
| | Known bugs, drift, historical notes (Gotchas & current state) |

**Linter config.** Make sure no linter *requires* docblocks, and let the formatter strip redundant ones:

| Stack | Config |
|---|---|
| PHP (Laravel Pint / PHP-CS-Fixer) | `pint.json`: `{"preset": "laravel", "rules": {"no_superfluous_phpdoc_tags": {"allow_mixed": true, "remove_inheritdoc": true}, "no_empty_phpdoc": true, "phpdoc_trim": true}}` |
| JS / TS (ESLint) | Do not enable `jsdoc/require-jsdoc`; type information belongs in TypeScript types |
| Java (Checkstyle) | Disable `MissingJavadocMethod` and `MissingJavadocType` |
| Other | Disable any rule that makes doc comments mandatory |

**Post-task routine.** The shared rules loaded by each agent entrypoint tell the agent to run this after every task that changes code, when the local switch (§5.0) is `on`. It uses the code-only update, so the graph step needs no LLM. Tokens are spent only on writing the notes of the nodes that were touched.

```bash
PY=$(cat graphify-out/.graphify_python)
graphify update .                                              # code only, no LLM
cp graphify-out/graph.json graphify-out/graph.pre-curation.json
$PY graphify-out/curate.py && $PY graphify-out/install_curated.py
# check the communities install_curated.py printed (see below); fix community_names.json and re-run install
$PY graphify-out/graphify_tools.py missing                     # new nodes → write their notes
# update notes of changed nodes in node_notes.json (Why / Gotchas for decisions and bugs from this task)
$PY graphify-out/graphify_tools.py where-lines                 # refresh line numbers in Where (no LLM)
graphify export obsidian --dir docs/graphify && $PY graphify-out/enrich_vault.py docs/graphify
```

**Check the community names every time.** Communities are recomputed on each install, and `community_names.json` maps the label of a community's most-connected node to its name. New or merged nodes shift both the members and that top label, so a name can silently disappear (the community is shown under a node label such as `user.service.ts`) or stick to a community whose members it no longer describes. After `install_curated.py`, compare every printed community with its members in `graphify-out/graph.json`; when one is unnamed or misnamed, rewrite `community_names.json` for the current top labels (§4.5) and run `install_curated.py` again. Clustering is deterministic for the same graph, so the names then stay put until the graph changes.

**Write the Where section with line numbers.** For a class, the first bullet is `path:L<line>` followed by its methods in line order, e.g. ``- `src/modules/users/services/user.service.ts:L31` (register L42, verifyEmail L77)``; then related code, then docs. Take the numbers from the node's `source_location` and `members` in `graph.json`, not from memory. This applies to notes written directly into `node_notes.json` as well as to batch notes (`notes_work/PROMPT.md`). `graphify_tools.py where-lines` rewrites that first bullet from `graph.json` on every render, so line numbers never go stale when code moves; it touches only the bullet that names the node's own `source_file` and leaves the rest of the note alone.

**Why start this at init.** Coding agents copy the comment density of the code around them. A codebase that starts clean stays clean, while one that already has heavy comments keeps attracting more. Adding the rule later means cleaning up the existing comments first.

---

## 6. Daily use: reading the project

| Need | Command |
|---|---|
| "How does X work?" | `graphify query "how does the refresh token work?"` |
| How A relates to B | `graphify path "OrderService" "PaymentGateway"` |
| One node and its neighbours | `graphify explain "AuthService"` |
| Full detail | open `docs/graphify/<Name>.md` |
| Big picture | `graphify-out/GRAPH_REPORT.md` or `graphify-out/graph.html` |
| Visual | Obsidian Graph view on `docs/graphify` |

Recommended order: `query`/`explain` to find the node, then read its note (especially **Gotchas & current state**), then open the code listed under **Where**.

---

## 7. Maintenance: after changing code

This section applies when the local switch (§5.0) is `on`.

Default to code-only refresh. Only when full semantic extraction is requested, use the current host's skill or a configured CLI backend (§2.2) before the same curation/rendering steps.

```bash
graphify update .              # CLI: code only, no LLM; add --force after intentional code deletions

PY=$(cat graphify-out/.graphify_python)
cp graphify-out/graph.json graphify-out/graph.pre-curation.json
$PY graphify-out/curate.py && $PY graphify-out/install_curated.py
# check and fix the community names (§5.3, "Check the community names every time")
$PY graphify-out/graphify_tools.py missing          # new nodes without a note
```

Then save the context into the notes:
- **New nodes** (a new class, endpoint or migration): write notes with `batches` → delegated or sequential processing → `merge-notes`, or write them directly into `node_notes.json`.
- **Changed nodes**: a new method appears under *Members* automatically. The explanation, rationale and gotchas must be updated by hand in that node's entry in `node_notes.json`.
- **Decisions, bugs and trade-offs**: put them under `## Why` or `## Gotchas & current state` in the note of the most relevant node.
- **New duplicates**, usually from docs: add entries to `curation_map.json` and re-run `curate.py` + `install_curated.py`.
- **Merged nodes**: when a file node is folded into its class (for example after adjusting `CODE_EXTS` handling), move anything useful from the file's note into the class note and delete the file's entry from `node_notes.json`; the old id no longer exists.

Render and commit:
```bash
$PY graphify-out/graphify_tools.py where-lines
graphify export obsidian --dir docs/graphify
$PY graphify-out/enrich_vault.py docs/graphify
graphify export html
git add docs/graphify graphify-out/node_notes.json graphify-out/curation_map.json graphify-out/community_names.json
```

**Prompt:**
> I changed the code. Refresh the graph following docs/GRAPHIFY.md §7: update, curate, install, then list new nodes without notes and existing nodes whose source files changed in this diff. Write or update only those notes, render the vault, and show me what changed.

---

## 8. Gotchas

- **Agent host and LLM backend are different settings.** Install the skill for the host, configure a provider only for semantic CLI extraction, and share the same graph/notes across model changes.
- **Instruction files are host-specific.** Keep the workflow in `docs/GRAPHIFY_RULES.md` and explicitly load it through each host's entrypoint; do not assume `CLAUDE.md`, `@file` imports, slash commands, or subagents work everywhere.

- **Re-read shared files before writing.** Several agents can work in one repo at once, and `node_notes.json`, `community_names.json` and `curation_map.json` are rewritten whole, so the last writer silently wins. Read right before changing, write only your entries, and write back at once. `graphify_tools.py` refuses to save `node_notes.json` when it changed after being read and writes atomically; scripts you write yourself must do the same.
- **Don't hand-edit `docs/graphify/*.md`.** Edit `node_notes.json`; the next export overwrites the vault.
- **Never regenerate all notes at once.** Write or update only new and changed nodes.
- **Renaming or moving a class changes its node id**, because code ids are derived from path + name. The old note is then orphaned. `graphify_tools.py missing` shows the new id, and you move the note entry to it.
- **Concept nodes get a stable id** from their `curation_map.json` key, so their notes survive re-runs. Pick keys carefully and don't rename them later.
- **Keep the vault and `graphify-out/` in `.graphifyignore`**, or the graph ingests its own output.
- **Keep temp files inside the repo** (`notes_work/tmp_NN/`). A reboot can wipe `/tmp` and lose finished batches.
- **Pin the graphify version.** `enrich_vault.py` relies on internal helpers of `graphify.export`.
- **Notes use bold names, not `[[wikilinks]]`.** Links come only from the Connections list, so Obsidian's graph reflects real edges.
- **A note can only be written once its node exists.** Note ids are graph node ids, so a new class needs the post-task refresh (§5.3) before its note can be added. Until then, keep the context in the task conversation, not in a code comment.
- **Keep empty notes.** An empty note (`""`) marks scaffolding that intentionally has no note. Deleting the entry does not break the vault, but the node then shows up in `missing` on every refresh and invites notes nobody needs.
- **`where-lines` only refreshes a bullet it recognises**: the first Where bullet that is exactly the node's `source_file`, optionally with `:L<n>` and a `(method L<n>, ...)` list. A bullet with extra text after it, a class whose note points at another file first, or a doc node is left as written. Methods come from *Members*, so a method folded elsewhere does not appear.
- **Community names drift.** A name is keyed by the top node label of its community, which changes when the graph changes. Check the names after every install (§5.3).
- **File nodes only merge into a class they are named after.** Kebab-case stacks (NestJS, Angular) need the `kebab-case.role.<ext>` rule in `curate.py`, otherwise every class gets a separate "File holding X" node.
- **graphify reads comments as `rationale`.** With few comments the graph has fewer rationale facts, which is expected, because the rationale lives in the notes instead.

---

## Appendix

### A. `graphify-out/curate.py`

```python
"""Curate the raw graphify graph: fold low-level nodes into their owners and merge duplicates.

Input:  graphify-out/graph.pre-curation.json (falls back to graph.json)
Output: graphify-out/graph.curated.json (install_curated.py turns it into graph.json)

    $(cat graphify-out/.graphify_python) graphify-out/curate.py [--phase1-only]

Phase 1 (rule-based): framework/vendor symbols removed (kept as text on the nodes that used
    them), methods -> owning class, file-level functions -> file node, `X.<ext>` file node ->
    class X defined in it, manifest keys (package.json, composer.json, ...) -> manifest node.
Phase 2 (mapping): graphify-out/curation_map.json, written once by an LLM review, folds or
    merges document-derived duplicates. Merged nodes keep their history in `aliases`,
    `sources`, `members`, `external_refs` and `rationales`.

Edit the CONFIG block for your stack; everything below it is generic.
"""
import json
import re
import sys
from collections import defaultdict
from pathlib import Path

import networkx as nx

# ---------------------------------------------------------------- CONFIG (edit per project)
# Framework / vendor symbol prefixes that should not become nodes of their own.
EXTERNAL_RX = re.compile(r"^(Illuminate|Symfony|PHPUnit|java\.|javax\.|org\.springframework|"
                         r"@nestjs|express|react|lodash|self$|static$|parent$|this$|Closure$)")
# Path prefixes that hold project code (anything else is treated as docs).
CODE_ROOTS = ("src/", "app/", "lib/", "tests/", "test/", "config/", "routes/", "database/")
# Source file extensions whose file node is merged into the class it is named after (see class_candidates).
CODE_EXTS = (".php", ".java", ".kt", ".ts", ".tsx", ".js", ".py", ".go", ".rb", ".cs")
# Small file-level symbols that fold into their file node instead of becoming nodes of their own:
# constants and enum members (UPPER_CASE) and framework page exports such as Next.js metadata and fonts.
SMALL_SYMBOL_RX = re.compile(r"^([A-Z][A-Z0-9_]*[A-Z0-9]|metadata|inter|geistSans|geistMono)$")
# Test files whose inner symbols (helpers, fixtures, probe classes) fold into the test file node.
TEST_FILE_RX = re.compile(r"\.(spec|e2e-spec|test)\.[jt]sx?$")
# Manifest files whose keys (dependencies, scripts) fold into the manifest node.
MANIFESTS = ("package.json", "composer.json", "pyproject.toml", "pom.xml", "build.gradle", "go.mod")
# Doc path prefixes, most authoritative first; used to pick the surviving node of a duplicate group.
DOC_PRIORITY = ("docs/", "README.md")
# Optional hub nodes that group files nothing else references:
#   {"hub-id": ("Hub Label", lambda source_file: bool)}
HUBS = {
    # "config-files": ("Config Files", lambda sf: sf.startswith("config/")),
}
# Optional direct links for files that would otherwise be orphans: {"path/file": ["node_id", ...]}
USED_BY = {}
# ---------------------------------------------------------------- end CONFIG

OUT = Path(__file__).resolve().parent
REDIRECT = {}  # removed/renamed node id -> id it now lives in (for remapping hyperedges)
CONF_RANK = {"EXTRACTED": 3, "INFERRED": 2, "AMBIGUOUS": 1}


def load(path):
    data = json.loads(Path(path).read_text(encoding="utf-8"))
    G = nx.Graph()
    for n in data["nodes"]:
        G.add_node(n["id"], **{k: v for k, v in n.items() if k != "id"})
    for e in data["links"]:
        add_edge(G, e["source"], e["target"], {k: v for k, v in e.items() if k not in ("source", "target")})
    return data, G


def add_edge(G, u, v, attrs):
    if u == v or u not in G or v not in G:
        return
    if G.has_edge(u, v):
        old = G.edges[u, v]
        if CONF_RANK.get(attrs.get("confidence"), 0) <= CONF_RANK.get(old.get("confidence"), 0):
            return
    G.add_edge(u, v, **attrs)


def _append(d, key, values):
    cur = d.setdefault(key, [])
    for v in values:
        if v and v not in cur:
            cur.append(v)


def merge(G, src, dst, as_member=False):
    """Fold node `src` into `dst`, moving its edges and keeping its identity as history."""
    if src == dst or src not in G or dst not in G:
        return
    s, d = G.nodes[src], G.nodes[dst]
    loc = f"{s.get('source_file') or ''}:{s.get('source_location') or ''}".rstrip(":")
    if as_member:
        _append(d, "members", [s.get("label", src) + (f" ({s['source_location']})" if s.get("source_location") else "")])
    else:
        _append(d, "aliases", [s.get("label", src)] + s.get("aliases", []))
    _append(d, "sources", [loc] + s.get("sources", []))
    for key in ("members", "external_refs"):
        _append(d, key, s.get(key, []))
    if s.get("rationale"):
        _append(d, "rationales", [s["rationale"]])
    _append(d, "rationales", s.get("rationales", []))
    for nb in list(G.neighbors(src)):
        if nb != dst:
            add_edge(G, dst, nb, dict(G.edges[src, nb]))
    G.remove_node(src)
    REDIRECT[src] = dst


def class_candidates(filename):
    """Class names a source file may be named after, most specific first."""
    stem = filename.rsplit(".", 1)[0]
    pascal = lambda words: "".join(w[:1].upper() + w[1:] for w in words if w)
    words = re.split(r"[.\-_]", stem)
    names = [stem, pascal(words)]
    if len(words) > 1:
        names.append(pascal(words[:-1]))
    migration = re.match(r"^(\d+)-(\w+)$", stem)
    if migration:
        names.append(migration.group(2) + migration.group(1))
    return names


def phase1(G):
    lab = lambda n: G.nodes[n].get("label", n)
    sf = lambda n: G.nodes[n].get("source_file") or ""
    stats = defaultdict(int)

    # 1. framework / vendor symbols: record on neighbours, then drop
    for n in [n for n in G if G.nodes[n].get("file_type") == "code" and EXTERNAL_RX.match(lab(n))]:
        for nb in G.neighbors(n):
            _append(G.nodes[nb], "external_refs", [f"{G.edges[n, nb].get('relation', '')} {lab(n)}".strip()])
        G.remove_node(n)
        stats["external removed"] += 1

    # 2. methods -> owning class
    for u, v, a in list(G.edges(data=True)):
        if a.get("relation") != "method" or u not in G or v not in G:
            continue
        cls, m = (u, v) if not lab(u).startswith(".") else (v, u)
        merge(G, m, cls, as_member=True)
        stats["methods folded"] += 1

    # 2b. methods with no `method` edge -> class in the same file
    for n in [n for n in G if lab(n).startswith(".") and sf(n).startswith(CODE_ROOTS)]:
        cls = [c for c in G if c != n and sf(c) == sf(n) and G.nodes[c].get("_callable_class")]
        if cls:
            merge(G, n, cls[0], as_member=True)
            stats["methods folded"] += 1

    # 3. file-level functions -> file node
    for n in [n for n in G if lab(n).endswith("()") and sf(n).startswith(CODE_ROOTS)]:
        files = [nb for nb in G.neighbors(n) if lab(nb) == Path(sf(n)).name]
        if files:
            merge(G, n, files[0], as_member=True)
            stats["file functions folded"] += 1

    # 3b. small symbols (constants, page metadata/fonts) and everything inside test files -> file node
    file_node = {sf(f): f for f in G if sf(f).startswith(CODE_ROOTS) and lab(f) == Path(sf(f)).name}
    for n in [n for n in list(G) if sf(n).startswith(CODE_ROOTS)]:
        owner = file_node.get(sf(n))
        if n in G and owner and n != owner and (SMALL_SYMBOL_RX.match(lab(n)) or TEST_FILE_RX.search(sf(n))):
            merge(G, n, owner, as_member=True)
            stats["small symbols folded into file"] += 1

    # 4. file node -> the class it is named after (Name.<ext>, kebab-case.role.<ext>, <timestamp>-Name.<ext>)
    for f in [n for n in G if lab(n).endswith(CODE_EXTS)]:
        in_file = {lab(n): n for n in G if n != f and sf(n) == sf(f)}
        match = next((in_file[c] for c in class_candidates(Path(lab(f)).name) if c in in_file), None)
        if match:
            merge(G, f, match)
            stats["file nodes merged into class"] += 1

    # 5. manifest keys -> the manifest file node
    for manifest in MANIFESTS:
        roots = [n for n in G if lab(n) == manifest and sf(n) == manifest]
        if roots:
            for n in [n for n in G if sf(n) == manifest and n != roots[0]]:
                merge(G, n, roots[0], as_member=True)
                stats[f"{manifest} keys folded"] += 1
    return stats


def phase2(G, mapping):
    """mapping: {node_id: {"action": "fold", "into": id} | {"action": "merge", "key": k, "label"?: l}}"""
    groups = defaultdict(list)
    folded = 0
    for nid, m in mapping.items():
        if nid not in G:
            continue
        if m.get("action") == "fold" and m.get("into") in G:
            merge(G, nid, m["into"], as_member=True)
            folded += 1
        elif m.get("action") == "merge" and m.get("key"):
            groups[m["key"]].append(nid)
    merged = 0
    renames = {}
    for key, ids in groups.items():
        ids = [i for i in ids if i in G]
        # key may name an existing canonical node id directly (usually a code entity)
        if key in G and key not in ids:
            ids.insert(0, key)
        if not ids:
            continue
        canon = key if key in ids else pick_canonical(G, ids)
        labels = [mapping[i]["label"] for i in ids if mapping.get(i, {}).get("label")]
        for i in ids:
            if i != canon:
                merge(G, i, canon)
                merged += 1
        if not (G.nodes[canon].get("source_file") or "").startswith(CODE_ROOTS):
            if labels:
                G.nodes[canon]["label"] = max(set(labels), key=labels.count)
            # concept nodes get the curation key as a stable id, so written notes survive re-runs
            if canon != key and key not in G:
                renames[canon] = key
    nx.relabel_nodes(G, renames, copy=False)
    REDIRECT.update(renames)
    return {"doc nodes folded": folded, "duplicates merged": merged}


def link_orphans(G):
    for hub, (label, match) in HUBS.items():
        members = [n for n in G if match(G.nodes[n].get("source_file") or "")]
        if not members:
            continue
        if hub not in G:
            G.add_node(hub, label=label, file_type="concept", source_file="")
        for n in members:
            if n != hub:
                add_edge(G, hub, n, {"relation": "contains", "confidence": "EXTRACTED", "confidence_score": 1.0,
                                     "source_file": G.nodes[n].get("source_file"), "weight": 1.0})
    for n in list(G):
        for t in USED_BY.get(G.nodes[n].get("source_file") or "", []):
            if t in G and G.degree(n) < 3:
                add_edge(G, n, t, {"relation": "references", "confidence": "INFERRED", "confidence_score": 0.95,
                                   "source_file": G.nodes[n].get("source_file"), "weight": 1.0})


def disambiguate_labels(G):
    """Give nodes that still share a label a distinguishing suffix (vault filenames come from labels)."""
    by_label = defaultdict(list)
    for n in G:
        by_label[G.nodes[n].get("label", n).lower()].append(n)
    for ids in by_label.values():
        if len(ids) < 2:
            continue
        for n in ids:
            d = G.nodes[n]
            src = d.get("source_file") or ""
            if src.startswith(CODE_ROOTS):
                d["label"] = f"{d['label']} ({Path(src).parent.name})"
            elif src:
                d["label"] = f"{d['label']} ({Path(src).stem})"


def resolve(nid):
    seen = set()
    while nid in REDIRECT and nid not in seen:
        seen.add(nid)
        nid = REDIRECT[nid]
    return nid


def remap_hyperedges(hyperedges, G):
    out, seen = [], set()
    for h in hyperedges:
        nodes = list(dict.fromkeys(resolve(n) for n in h.get("nodes", [])))
        nodes = [n for n in nodes if n in G]
        sig = (h.get("label"), tuple(sorted(nodes)))
        if len(nodes) >= 3 and sig not in seen:
            seen.add(sig)
            out.append({**h, "nodes": nodes})
    return out


def pick_canonical(G, ids):
    def rank(i):
        d = G.nodes[i]
        s = d.get("source_file") or ""
        if s.startswith(CODE_ROOTS) and d.get("file_type") == "code":
            return 0
        for r, prefix in enumerate(DOC_PRIORITY, start=1):
            if s.startswith(prefix):
                return r
        return len(DOC_PRIORITY) + 1
    return sorted(ids, key=lambda i: (rank(i), -G.degree(i)))[0]


def main():
    raw = OUT / "graph.pre-curation.json"
    src = raw if raw.exists() else OUT / "graph.json"
    data, G = load(src)
    before = (G.number_of_nodes(), G.number_of_edges())
    stats = dict(phase1(G))
    map_path = OUT / "curation_map.json"
    if map_path.exists() and "--phase1-only" not in sys.argv:
        stats.update(phase2(G, json.loads(map_path.read_text(encoding="utf-8"))))
    link_orphans(G)
    disambiguate_labels(G)
    for n in [n for n in G if G.degree(n) == 0 and G.nodes[n].get("file_type") == "code"
              and not (G.nodes[n].get("source_file") or "")]:
        G.remove_node(n)
    nodes = [{"id": n, **a} for n, a in G.nodes(data=True)]
    links = [{"source": u, "target": v, **a} for u, v, a in G.edges(data=True)]
    out = {k: v for k, v in data.items() if k not in ("nodes", "links")}
    out["hyperedges"] = remap_hyperedges(data.get("hyperedges", []), G)
    out.update(nodes=nodes, links=links)
    (OUT / "graph.curated.json").write_text(json.dumps(out, ensure_ascii=False), encoding="utf-8")
    print(f"input:  {src.name}")
    print(f"before: {before[0]} nodes, {before[1]} edges")
    for k, v in stats.items():
        print(f"  {k}: {v}")
    print(f"after:  {G.number_of_nodes()} nodes, {G.number_of_edges()} edges -> graph.curated.json")


if __name__ == "__main__":
    main()
```

### B. `graphify-out/install_curated.py`

```python
"""Install graph.curated.json as graph.json: cluster, name communities, regenerate GRAPH_REPORT.md.

Keeps the raw (uncurated) graph as graph.pre-curation.json so curate.py can always start
from it. Community names come from community_names.json ({top node label: name}); a
community whose most-connected node isn't listed is named after that node.

Full refresh after `/graphify . --update`:
    PY=$(cat graphify-out/.graphify_python)
    cp graphify-out/graph.json graphify-out/graph.pre-curation.json   # the fresh raw graph
    $PY graphify-out/curate.py && $PY graphify-out/install_curated.py
    graphify export obsidian --dir docs/graphify && $PY graphify-out/enrich_vault.py docs/graphify
    graphify export html
"""
import json
from pathlib import Path

import networkx as nx
from graphify.analyze import god_nodes, suggest_questions, surprising_connections
from graphify.cluster import cluster, score_all
from graphify.detect import detect
from graphify.export import to_json
from graphify.report import generate

OUT = Path(__file__).resolve().parent


def main():
    d = json.loads((OUT / "graph.curated.json").read_text(encoding="utf-8"))
    G = nx.Graph()
    for n in d["nodes"]:
        G.add_node(n["id"], **{k: v for k, v in n.items() if k not in ("id", "community", "community_name")})
    for e in d["links"]:
        G.add_edge(e["source"], e["target"], **{k: v for k, v in e.items() if k not in ("source", "target")})
    G.graph["hyperedges"] = d.get("hyperedges", [])

    comms = cluster(G)
    cohesion = score_all(G, comms)
    names_path = OUT / "community_names.json"
    names = json.loads(names_path.read_text(encoding="utf-8")) if names_path.exists() else {}
    labels = {}
    for cid, members in comms.items():
        top = G.nodes[max(members, key=G.degree)]["label"]
        labels[cid] = names.get(top, top)
    to_json(G, comms, str(OUT / "graph.json"), force=True, community_labels=labels)
    detection = detect(OUT.parent)
    report = generate(G, comms, cohesion, labels, god_nodes(G), surprising_connections(G, comms), detection,
                      {"input": 0, "output": 0}, ".", suggested_questions=suggest_questions(G, comms, labels))
    (OUT / "GRAPH_REPORT.md").write_text(report, encoding="utf-8")
    (OUT / ".graphify_labels.json").write_text(json.dumps({str(k): v for k, v in labels.items()}), encoding="utf-8")
    print(f"installed: {G.number_of_nodes()} nodes, {G.number_of_edges()} edges, {len(comms)} communities")
    for cid in sorted(comms, key=lambda c: -len(comms[c])):
        print(f"  {labels[cid]} ({len(comms[cid])})")


if __name__ == "__main__":
    main()
```

### C. `graphify-out/enrich_vault.py`

```python
"""Add full notes to the Obsidian vault written by `graphify export obsidian`.

graphify's vault notes only carry frontmatter + a Connections list. This script inserts,
under each note's title:
  - the written note from graphify-out/node_notes.json ({node_id: markdown}), and
  - the curation facts on the node (merged aliases, source locations, members such as
    class methods, framework dependencies), so nothing folded away by curate.py is lost.

Run after every `graphify export obsidian --dir <vault>`:
    $(cat graphify-out/.graphify_python) graphify-out/enrich_vault.py <vault_dir>
"""
import json
import sys
from pathlib import Path

import networkx as nx
from graphify.export import _dedup_node_filenames, _obsidian_safe_stem, stem_filename_budget, _DEDUP_SUFFIX_RESERVE

OUT = Path(__file__).resolve().parent
MARK_START, MARK_END = "<!-- graphify-notes:start -->", "<!-- graphify-notes:end -->"


def facts(d):
    parts = []
    srcs = [s for s in d.get("sources", []) if s]
    main = d.get("source_file")
    if main or srcs:
        where = [f"`{main}{':' + d['source_location'] if d.get('source_location') else ''}`"] if main else []
        where += [f"`{s}`" for s in srcs if not main or not s.startswith(main)]
        parts.append("**Sources:** " + ", ".join(dict.fromkeys(where)))
    label = d.get("label", "")
    aliases = [a for a in dict.fromkeys(d.get("aliases", [])) if a not in (label, f"{label}.php") and not a.endswith(f"/{label}.php")]
    if aliases:
        parts.append("**Also known as:** " + ", ".join(aliases))
    if d.get("members"):
        parts.append("**Members:**\n" + "\n".join(f"- `{m}`" for m in d["members"]))
    if d.get("external_refs"):
        ext = sorted({r.split(" ", 1)[-1] for r in d["external_refs"]})
        parts.append("**Framework dependencies:** " + ", ".join(f"`{e}`" for e in ext))
    return "\n\n".join(p for p in parts if not p.endswith(": "))


def main(vault):
    vault = Path(vault)
    data = json.loads((OUT / "graph.json").read_text(encoding="utf-8"))
    G = nx.Graph()
    for n in data["nodes"]:
        G.add_node(n["id"], **{k: v for k, v in n.items() if k != "id"})
    limit = stem_filename_budget(vault, reserve=_DEDUP_SUFFIX_RESERVE)
    fname = _dedup_node_filenames(G, lambda label: _obsidian_safe_stem(label, limit))
    notes_path = OUT / "node_notes.json"
    notes = json.loads(notes_path.read_text(encoding="utf-8")) if notes_path.exists() else {}
    done = missing = 0
    for nid, d in G.nodes(data=True):
        f = vault / f"{fname[nid]}.md"
        if not f.exists():
            missing += 1
            continue
        text = f.read_text(encoding="utf-8")
        if MARK_START in text:  # re-run: replace the previous block
            text = text[: text.index(MARK_START)] + text[text.index(MARK_END) + len(MARK_END):].lstrip("\n")
        body = "\n\n".join(p for p in (notes.get(nid, "").strip(), facts(d)) if p)
        if not body:
            continue
        title = f"# {d.get('label', nid)}\n"
        i = text.find(title)
        if i < 0:
            continue
        i += len(title)
        text = text[:i] + f"\n{MARK_START}\n{body}\n{MARK_END}\n" + text[i:]
        f.write_text(text, encoding="utf-8")
        done += 1
    print(f"enriched {done} notes ({len(notes)} written notes available, {missing} node files not found)")


if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "docs/graphify")
```

### D. `graphify-out/graphify_tools.py`

```python
"""Helpers for the note-writing and curation loop of the curated graphify vault.

    PY=$(cat graphify-out/.graphify_python)
    $PY graphify-out/graphify_tools.py missing               # node ids in graph.json without a written note
    $PY graphify-out/graphify_tools.py doc-nodes             # non-code nodes -> notes_work/doc_nodes.json (curation review input)
    $PY graphify-out/graphify_tools.py batches [--size 28] [--all]   # nodes without notes -> notes_work/nbatch_NN.json
    $PY graphify-out/graphify_tools.py merge-notes [--overwrite]     # notes_work/notes_NN.json -> node_notes.json
    $PY graphify-out/graphify_tools.py baseline              # mark every node without a note as "no note needed"
    $PY graphify-out/graphify_tools.py where-lines           # refresh `path:L<n> (method L<n>, ...)` in each note's Where

Commands that change node_notes.json re-read it, refuse to write when the file changed after it was
read (another agent or process), and write atomically through a temporary file.

An empty note ("") means the node is framework boilerplate that intentionally has no written note:
`missing` skips it and enrich_vault.py still renders its facts (Sources, Members). Replace the empty
string with a real note as soon as the node is changed by project work.
"""
import argparse
import json
import os
import re
from pathlib import Path

OUT = Path(__file__).resolve().parent
WORK = OUT / "notes_work"


def read(path, default=None):
    return json.loads(path.read_text(encoding="utf-8")) if path.exists() else default


def graph():
    return read(OUT / "graph.json")


NOTES = OUT / "node_notes.json"
_read_stamp = {}


def notes():
    _read_stamp[NOTES] = NOTES.stat().st_mtime_ns if NOTES.exists() else None
    return read(NOTES, {})


def save_notes(data):
    now = NOTES.stat().st_mtime_ns if NOTES.exists() else None
    if NOTES in _read_stamp and now != _read_stamp[NOTES]:
        raise SystemExit("node_notes.json changed after it was read (another agent or process?). "
                         "Nothing was written; run the command again.")
    tmp = NOTES.with_suffix(".json.tmp")
    tmp.write_text(json.dumps(data, ensure_ascii=False, indent=1), encoding="utf-8")
    os.replace(tmp, NOTES)


def neighbors(g):
    label = {n["id"]: n.get("label", n["id"]) for n in g["nodes"]}
    out = {}
    for e in g["links"]:
        rel = e.get("relation", "related")
        out.setdefault(e["source"], []).append(f"{rel} -> {label.get(e['target'], e['target'])}")
        out.setdefault(e["target"], []).append(f"{rel} <- {label.get(e['source'], e['source'])}")
    return out


def describe(n, nbs):
    return {
        "id": n["id"],
        "label": n.get("label", n["id"]),
        "type": n.get("file_type"),
        "community": n.get("community_name", n.get("community")),
        "source_file": n.get("source_file"),
        "source_location": n.get("source_location"),
        "other_sources": n.get("sources", []),
        "aliases": n.get("aliases", []),
        "members": n.get("members", []),
        "rationales": n.get("rationales", []),
        "framework_refs": n.get("external_refs", []),
        "neighbors": nbs.get(n["id"], [])[:40],
    }


def cmd_missing(_):
    have = notes()
    missing = [n["id"] for n in graph()["nodes"] if n["id"] not in have]
    print(f"{len(missing)} nodes without a note")
    for nid in missing:
        print(" ", nid)


def cmd_doc_nodes(_):
    g = read(OUT / "graph.pre-curation.json") or graph()  # review the raw graph, before merges
    nbs = neighbors(g)
    rows = [describe(n, nbs) for n in g["nodes"] if n.get("file_type") != "code"]
    WORK.mkdir(exist_ok=True)
    (WORK / "doc_nodes.json").write_text(json.dumps(rows, ensure_ascii=False, indent=1), encoding="utf-8")
    print(f"{len(rows)} non-code nodes -> {WORK / 'doc_nodes.json'}")


def cmd_batches(a):
    g, have = graph(), notes()
    nbs = neighbors(g)
    todo = [describe(n, nbs) for n in g["nodes"] if a.all or n["id"] not in have]
    todo.sort(key=lambda d: (str(d["community"]), d["source_file"] or ""))  # keep related nodes together
    WORK.mkdir(exist_ok=True)
    for old in WORK.glob("nbatch_*.json"):
        old.unlink()
    for i in range(0, len(todo), a.size):
        path = WORK / f"nbatch_{i // a.size + 1:02d}.json"
        path.write_text(json.dumps(todo[i:i + a.size], ensure_ascii=False, indent=1), encoding="utf-8")
    print(f"{len(todo)} nodes -> {-(-len(todo) // a.size)} batches in {WORK}")


def cmd_merge_notes(a):
    have = notes()
    ids = {n["id"] for n in graph()["nodes"]}
    added = replaced = 0
    for path in sorted(WORK.glob("notes_*.json")):
        for nid, text in read(path, {}).items():
            if nid not in ids or not text.strip():
                continue
            if nid in have and not a.overwrite:
                continue
            replaced += nid in have
            added += nid not in have
            have[nid] = text
    save_notes(have)
    print(f"node_notes.json: {added} added, {replaced} replaced, {len(ids - set(have))} nodes still without a note")


def cmd_baseline(_):
    have = notes()
    new = {n["id"]: "" for n in graph()["nodes"] if n["id"] not in have}
    have.update(new)
    save_notes(have)
    print(f"baseline: {len(new)} nodes marked as intentionally without a note")


def where_line(n):
    methods = []
    for m in n.get("members", []):
        hit = re.match(r"^\.(\w+)\(\)\s*\(L(\d+)\)$", m)
        if hit and hit.group(1) != "constructor":
            methods.append((int(hit.group(2)), hit.group(1)))
    suffix = f" ({', '.join(f'{name} L{line}' for line, name in sorted(methods))})" if methods else ""
    return f"- `{n['source_file']}:{n['source_location']}`{suffix}"


def cmd_where_lines(_):
    nodes = {n["id"]: n for n in graph()["nodes"]}
    have = notes()
    changed = 0
    for nid, text in have.items():
        n = nodes.get(nid)
        if not text or not n or n.get("file_type") != "code" or not n.get("source_file") or not n.get("source_location"):
            continue
        own = re.compile(r"^- `" + re.escape(n["source_file"]) + r"(?::[^`]*)?`(?: \([^)]*\))?$")
        lines = text.split("\n")
        if "## Where" not in lines:
            continue
        for i in range(lines.index("## Where") + 1, len(lines)):
            if lines[i].startswith("## ") or lines[i].startswith("**"):
                break
            if own.match(lines[i]):
                fresh = where_line(n)
                if lines[i] != fresh:
                    lines[i] = fresh
                    changed += 1
                break
        have[nid] = "\n".join(lines)
    save_notes(have)
    print(f"where-lines: {changed} notes updated")


def main():
    p = argparse.ArgumentParser()
    sub = p.add_subparsers(dest="cmd", required=True)
    sub.add_parser("missing").set_defaults(fn=cmd_missing)
    sub.add_parser("doc-nodes").set_defaults(fn=cmd_doc_nodes)
    b = sub.add_parser("batches")
    b.add_argument("--size", type=int, default=28)
    b.add_argument("--all", action="store_true", help="include nodes that already have a note")
    b.set_defaults(fn=cmd_batches)
    m = sub.add_parser("merge-notes")
    m.add_argument("--overwrite", action="store_true", help="replace existing notes with the batch output")
    m.set_defaults(fn=cmd_merge_notes)
    sub.add_parser("baseline").set_defaults(fn=cmd_baseline)
    sub.add_parser("where-lines").set_defaults(fn=cmd_where_lines)
    a = p.parse_args()
    a.fn(a)


if __name__ == "__main__":
    main()
```

### E. `graphify-out/notes_work/PROMPT.md` (note-writing prompt for batch agents)

Replace the `<…>` placeholders before running the batches.

```markdown
You write knowledge-base notes for a curated knowledge graph of the repository "<PROJECT NAME>":
<ONE PARAGRAPH: what the system does, the stack, the main modules>.
Repo root: <ABSOLUTE PATH TO REPO>

Your batch file (a JSON list of nodes) is given in your task. Each node has: id, label, type, community,
source_file, source_location, other_sources (file:line of doc/code mentions merged into it), aliases
(duplicate names merged into it), members (methods for classes, or folded sub-items for docs/concepts),
rationales, framework_refs, neighbors (relation -> label).

For EVERY node write one complete, accurate note in <LANGUAGE> Markdown. Read the actual sources: open the
node's source_file and the most relevant other_sources. For code, read the class; for docs, read the relevant
section. THE CODE IS THE SOURCE OF TRUTH: if docs and code disagree, describe what the code does and add a
short "Doc drift" line naming the doc that is wrong. Do not invent anything you did not read.

Verified facts you can rely on:
- <FACT 1: e.g. a header/enum mapping that many nodes mention>
- <FACT 2: e.g. a known bug, so every agent reports it the same way>
- <FACT 3: e.g. things that do NOT exist although some docs mention them>

Note format (no H1, the title already exists; use these H2 sections and skip a section only if it is truly empty):
## Overview
2–4 sentences: what it is and its role in the system.
## Details
The substance. Per kind:
- Class/service/controller/middleware/DTO/model: responsibility; each important method (from `members`) with
  one line on what it does; tables/columns or fields; errors it throws; auth/permissions; config it reads;
  what calls it and what it depends on.
- Migration/schema: table, columns and types, keys/FKs/unique constraints, which model uses it.
- Config file: which keys matter for this app and who reads them.
- Endpoint: method + path, middleware/permission, controller method, request fields, response shape, errors.
- Error/response code: code, messages, where it is thrown (or "not thrown — historical").
- Concept / design pattern / rule: how it works concretely in this codebase, with class/method names.
- Spec / change proposal: goal, what was implemented, key decisions and rejected alternatives, status.
- Document (README, guidelines, changelog, ...): purpose, main sections, how it is kept in sync.
Use bullet lists and short code spans; include small code snippets only when they clarify (max ~8 lines).
## Why
Design rationale and trade-offs, if any (use `rationales` and the docs).
## Gotchas & current state
Known bugs, doc drift, historical notes, things a developer must know. Omit if nothing.
## Where
Bullet list of file paths (with line numbers where known): code first, then docs.

Length: as long as needed to be complete, typically 150–450 words; small things (a config file with nothing
special, a trivial DTO) can be ~80 words. Refer to other nodes by their label in **bold** (do not use [[wikilinks]]).

Work efficiently: after every ~8 nodes, merge what you have into the output JSON file (read it if it exists,
add your new notes, write it back), so progress survives an interruption. If the output file already exists
when you start, keep its notes and only write the missing ids.

Output: a single JSON object {"<node id>": "<markdown note>", ...} with EVERY node id from your batch, at the
output path given in your task. Validate that it is parseable JSON with all ids before finishing. Reply with one
line: the number of notes written, plus any doc drift or bug you found that is NOT in the verified facts above.

IMPORTANT, isolation: several agents run in parallel. Put ANY helper or temporary files only inside your own
folder: graphify-out/notes_work/tmp_<your batch number>/. Never use a shared scratchpad, /tmp or generic names.
Only ever write to your own output path.
```
