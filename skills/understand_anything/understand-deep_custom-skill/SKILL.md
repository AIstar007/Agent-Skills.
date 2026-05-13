---
name: understand-deep
description: Build a deep knowledge graph with function-level control flow — capturing every if/else branch, loop, throw, and conditional call with zero LLM token waste on raw source reading
argument-hint: [options]
---

# /understand-deep

Analyze the current codebase and produce a `deep-knowledge-graph.json` file in `.understand-anything/`. Unlike `/understand`, every function node includes a deterministically extracted `controlFlow` profile — branches, loops, throws, and conditional vs unconditional calls — extracted by a script before any LLM token is spent.

## Design Principle

**Script first, LLM second.**
- The extraction script runs deterministically on raw source code → zero LLM tokens for structure
- The LLM only reads the compact JSON profile (≈80 tokens/function vs 200-500 for raw source)
- Control flow is captured by the script; the LLM uses it to write intelligent summaries

## Options

- `$ARGUMENTS` may contain:
  - `--full` — Force a full rebuild even if deep-knowledge-graph.json already exists
  - `--scope <dir>` — Scope analysis to a specific subdirectory
  - `--no-trivial` — Skip functions under 5 lines entirely (default: skip under 3 lines)

---

## Phase 0 — Pre-flight

1. Set `PROJECT_ROOT` to the current working directory.
2. Create output directories:
   ```bash
   mkdir -p $PROJECT_ROOT/.understand-anything/intermediate
   mkdir -p $PROJECT_ROOT/.understand-anything/tmp
   ```
3. Check if `$PROJECT_ROOT/.understand-anything/deep-knowledge-graph.json` already exists.
   - If it exists and `--full` is NOT in `$ARGUMENTS`, ask the user: "(a) rebuild, (b) do nothing?"
   - If they pick (b), STOP.

4. **Reuse existing scan if available:** Check if `$PROJECT_ROOT/.understand-anything/intermediate/scan-result.json` exists.
   - If YES: read it to get `$FILE_LIST` and `$IMPORT_MAP`. Skip Phase 1.
   - If NO: run Phase 1.

5. Collect project context:
   - Read `README.md` if present (first 3000 chars) → `$README_CONTENT`
   - Read primary manifest (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`) → `$MANIFEST_CONTENT`
   - Capture top-level tree:
     ```bash
     find $PROJECT_ROOT -maxdepth 2 -type f -not -path '*/node_modules/*' -not -path '*/.git/*' -not -path '*/dist/*' | head -100
     ```
     → `$DIR_TREE`

---

## Phase 1 — SCAN (only if scan-result.json does not exist)

Dispatch a subagent using the prompt template at `../understand/project-scanner-prompt.md`. Read the template file and pass its full content as the subagent's prompt, appending:

> **Additional context from main session:**
>
> Project README (first 3000 chars):
> ```
> $README_CONTENT
> ```
>
> Package manifest:
> ```
> $MANIFEST_CONTENT
> ```

Pass these parameters in the dispatch prompt:

> Scan this project directory to discover all project files (including non-code files like configs, docs, infrastructure), detect languages and frameworks.
> Project root: `$PROJECT_ROOT`
> Write output to: `$PROJECT_ROOT/.understand-anything/intermediate/scan-result.json`

After the subagent completes, read `scan-result.json`:
- Store file list as `$FILE_LIST` (with `fileCategory` per file)
- Store import map as `$IMPORT_MAP`

**Gate check:** If >200 files, inform the user and suggest scoping with `--scope`. Proceed only if user confirms.

---

## Phase 2 — DEEP ANALYZE

Batch `$FILE_LIST` into groups of **20-25 files each**.

**Batching rules:**
- Code files (`fileCategory: code`) are the primary target — batch them first
- Non-code files (config, docs, infra) can be mixed in to fill batches
- Keep related files together (same directory) when possible

For each batch, construct `batchImportData`:
```
batchImportData = {}
for each file in this batch:
  batchImportData[file.path] = $IMPORT_MAP[file.path] ?? []
```

Dispatch up to **5 subagents concurrently** using the prompt template at `./deep-file-analyzer-prompt.md`. Read the template file and pass its full content as the subagent's prompt, appending:

> **Additional context from main session:**
>
> Project: `<projectName>` — `<projectDescription>`
> Languages: `<languages from Phase 1>`

Fill in batch-specific parameters and dispatch:

> Analyze these files and produce GraphNode and GraphEdge objects with full control flow profiles.
> Project root: `$PROJECT_ROOT`
> Project: `<projectName>`
> Languages: `<languages>`
> Batch index: `<batchIndex>`
> Write output to: `$PROJECT_ROOT/.understand-anything/intermediate/deep-batch-<batchIndex>.json`
>
> Pre-resolved import data (use this for all import edge creation — do NOT re-resolve):
> ```json
> <batchImportData JSON>
> ```
>
> Files to analyze in this batch:
> 1. `<path>` (<sizeLines> lines, fileCategory: `<fileCategory>`)
> 2. `<path>` (<sizeLines> lines, fileCategory: `<fileCategory>`)
> ...

After **all batches** complete, read each `deep-batch-<N>.json` and merge:
- Combine all `nodes` arrays — on duplicate IDs, keep the later occurrence
- Combine all `edges` arrays — deduplicate by composite key `source + target + type`

---

## Phase 3 — ASSEMBLE

Merge all deep-batch results into one node+edge set. Integrity cleanup:

1. Remove any edge whose `source` or `target` references a node ID that does not exist
2. Remove duplicate node IDs (keep last occurrence)
3. Count: how many function nodes include a `controlFlow` field vs total function nodes

---

## Phase 4 — WRITE OUTPUT

Assemble the final JSON object and write it:

```json
{
  "version": "1.0.0",
  "type": "deep",
  "project": {
    "name": "<projectName>",
    "languages": ["<languages>"],
    "frameworks": ["<frameworks>"],
    "description": "<projectDescription>",
    "analyzedAt": "<ISO 8601 timestamp>"
  },
  "nodes": [<all merged nodes>],
  "edges": [<all merged edges>]
}
```

Write to: `$PROJECT_ROOT/.understand-anything/deep-knowledge-graph.json`

Report to the user:
- Total nodes (by type breakdown: file, function, class, config, document, service, etc.)
- Total edges (by type breakdown)
- Functions with full controlFlow profiles captured
- Files skipped
- Any errors
