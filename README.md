# Tim Notebook

**A CPU-first, fully local alternative to Google's NotebookLM.** Add your documents, chat with
them, get answers with clickable citations, and send your notes straight into your Obsidian
vault — powered by a local model through [Ollama](https://ollama.com) on a machine with **no GPU**.

> **Status: planning.** No application code yet. The complete design lives in
> **[PLAN.md](PLAN.md)** — architecture, data model, pipelines, performance tuning, milestones.
> This README is the summary; PLAN.md is the source of truth.

---

## What it does

| | |
|---|---|
| **Sources** | PDF, DOCX/PPTX/XLSX, Markdown, plain text, pasted text, URLs — with progress, cancel, retry, and resumable background ingestion |
| **Grounded chat** | Answers built only from *your* sources, with inline `[S1]` citations; click one to open the exact page and highlighted passage |
| **Verified citations** | Every marker is checked against the chunk it claims to support. Unsupported sentences are flagged, not hidden |
| **Digests** | 150–250 word summary, key topics and suggested questions per source, generated in the background |
| **Studio** | FAQ, study guide, briefing doc, timeline, mind map — with valid citations |
| **Notes** | Manual notes and "save this answer as a note"; export to Markdown or **straight into Obsidian** |
| **Search** | Hybrid keyword (FTS5) + semantic (vector) across every source, jumping to the passage |
| **Obsidian** | One click writes plain Markdown with YAML frontmatter and working wikilinks into your vault. Vault auto-detected, atomic writes, and re-export never clobbers an edit you made there |
| **Optional cloud** | When a 1.7B/4B model isn't enough, one click answers the same question with a free frontier model via [OmniRoute](https://github.com/cheaper-inference/OmniRoute) — **off by default**, per notebook, every answer labelled |

No Docker. No Postgres. No Node at runtime. One Python process, one SQLite file, one browser tab.

---

## Built for one specific machine

The design is tuned to a **Windows 11 laptop with 12 GB RAM, a 512 GB SSD and a 4-core 10th-gen
Intel CPU (no discrete GPU)** — ~15 W, likely single-channel memory, Intel iGPU only.

That hardware dictates the numbers. On CPU, *reading* context costs as much as writing the
answer, so at the generic settings an answer would take ~95 s. The plan cuts it to ~50 s by
rationing context instead of adding hardware:

| Knob | Generic | This machine |
|---|---|---|
| Retrieved chunks | 5 | **3** (~1.3k prompt tokens) |
| Answer length | 150 words | **120 words** |
| `num_ctx` | 4096 | **3072** |
| Chat model | — | **`qwen3:4b`**, with **`qwen3:1.7b`** as quick mode |
| **Full answer** | ~95 s | **~50 s (4B) / ~27 s (1.7B)** |

**Models.** `qwen3:4b` + `nomic-embed-text` + `qwen3:1.7b` ≈ 4.3 GB on disk, ~3.3 GB resident.
`qwen3:8b` is deliberately **not** a default here: with Windows 11 plus a browser resident it lands
right at the 12 GB edge and decodes at 2–3 tok/s.

→ Full reasoning: [PLAN.md §2.1–2.2](PLAN.md#21-hardware-profile-windows-11--12-gb--512-gb-ssd--4-core-10th-gen-intel) · [§9.4 latency budget](PLAN.md#94-latency-budget-this-machine-4-core-15-w-single-channel-assumed)

---

## Local by default

Designed to work with the network cable unplugged. Documents, chunks, embeddings, notes and the
search index stay on your machine. Online mode is a **per-notebook policy** (`local_only` by
default); even when enabled, only the 3–6 retrieved chunks for a single question are sent — never
a whole document, never the index. **Embeddings are always computed locally.**

**One honest exception: your own sync client.** Notes you export to Obsidian land in a vault that
syncs, so those notes live in your sync provider's cloud like every other note in that vault.
Inference stays local; the vault is the one online path the app uses by default. If a notebook
should not be in that cloud, point it at an unsynced vault.

→ [PLAN.md §10.1](PLAN.md#101-the-trade-stated-plainly) (cloud models) · [§11.5](PLAN.md#115-because-your-vault-syncs-the-default-configuration) (syncing vaults)

---

## How it works

```
┌──────────── browser (one tab) ─────────────┐
│ Sources │ Chat + citations │ Studio/Notes │
└────────────────────┬───────────────────────┘
                     │ HTTP + SSE (127.0.0.1:8765)
┌────────────────────▼───────────────────────┐
│ FastAPI · ingest queue · retrieval · answer │
│          INFERENCE GATE (priority queue)    │
└───────┬───────────────────────┬─────────────┘
        │                       │
  Ollama :11434           SQLite (WAL)
  chat + embed            FTS5 + sqlite-vec
        │
  OmniRoute :20128 (optional, off by default)
```

- **Hybrid retrieval** — FTS5 BM25 + vector KNN, fused with reciprocal rank, capped at 3 chunks.
- **One inference gate** — interactive chat preempts background work; the CPU is never split.
- **Citation verification** — token-overlap scoring per sentence, so a small model's bluffing is
  visible in the UI rather than plausible.
- **The vault is a publish target** — SQLite stays the source of truth; if the app disappears
  tomorrow, your notes still open in any editor.

→ [PLAN.md §5 architecture](PLAN.md#5-architecture) · [§6 data model](PLAN.md#6-data-model) · [§7 ingestion](PLAN.md#7-ingestion-pipeline) · [§8 retrieval](PLAN.md#8-retrieval-and-answering)

---

## Quick start

*(once v1 exists)*

```powershell
# 1) Models, smallest first (~4.3 GB total)
ollama pull nomic-embed-text    # embeddings
ollama pull qwen3:1.7b          # quick answers
ollama pull qwen3:4b            # real questions + digests

# 2) Ollama server settings, then QUIT THE TRAY APP and relaunch it
setx OLLAMA_NUM_PARALLEL 1
setx OLLAMA_MAX_LOADED_MODELS 2
setx OLLAMA_KEEP_ALIVE 30m
setx OLLAMA_CONTEXT_LENGTH 3072

# 3) The app
py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1
pip install fastapi uvicorn pymupdf4llm markitdown trafilatura sqlite-vec psutil httpx
python -m timnb                 # http://127.0.0.1:8765
```

Then, in the app: **Settings → Obsidian** to pick a vault, and **Settings → Free models → Connect
OmniRoute** if you want the optional cloud models — the app installs and starts the gateway itself.

**Read [PLAN.md §9.5](PLAN.md#95-windows-11-specifics-do-these-once-in-this-order) first.** Five
Windows-specific steps — env vars plus a tray restart, Defender exclusions, keeping the SQLite
database out of OneDrive, power mode, and a `run.bat` — that save hours of confusing symptoms.

Then work through the **[day-one checklist](PLAN.md#appendix-f--day-one-checklist-for-this-laptop-45-minutes-before-writing-code)**
(~45 min): it turns the estimates above into measurements on *your* hardware.

---

## Roadmap

| # | Milestone | Scope |
|---|---|---|
| M0 | Day-one setup + model bake-off | Env, models, and real tok/s numbers on this laptop |
| M1 | Vertical slice | Upload → chunk → retrieve → grounded answer with citations |
| M2 | Usable app | Job queue, 3-pane UI, streaming + stop, tier toggle, citation drawer, notes |
| M3 | Digests, studio & Obsidian | Studio artifacts, Markdown export, Obsidian vault export |
| M4 | Optional free cloud | One-click gateway connect, egress guard, attribution, audit log |
| M5 | Hardening | Eval harness, refusal threshold, OCR, packaging, backups |
| M6 | Audio (optional) | Local TTS narration of an existing summary |

v1 (M0–M3) ≈ **11–17 evenings** part-time; the cloud backend adds 4–5.
→ [PLAN.md §14 milestones](PLAN.md#14-milestones)

---

## Documentation map

Everything is in **[PLAN.md](PLAN.md)**:

| Section | Contents |
|---|---|
| [1. TL;DR](PLAN.md#1-tldr) | The plan in fifteen lines |
| [2. Hardware profile](PLAN.md#2-assumptions-correct-these-and-the-plan-changes) | What 12 GB and a 15 W CPU mean for the numbers |
| [3. Features](PLAN.md#3-what-like-notebooklm-actually-means) | v1 / v2 / v3 scope and explicit non-goals |
| [4. Build vs. adopt](PLAN.md#4-build-vs-adopt-decide-this-first) | Custom lean core vs. Open Notebook vs. SurfSense |
| [5–8. Design](PLAN.md#5-architecture) | Architecture, schema, ingestion, retrieval, citations |
| [9. CPU performance](PLAN.md#9-making-cpu-inference-feel-fast) | Ollama config, memory math, scheduling, Windows specifics |
| [10. Optional cloud](PLAN.md#10-optional-online-mode-free-models-via-omniroute) | OmniRoute, egress policies, one-click connect, trust features |
| [11. Obsidian](PLAN.md#11-obsidian-notes-land-in-your-vault) | Vault discovery, frontmatter, safe writes, syncing vaults |
| [12–13. UI & API](PLAN.md#12-ui) | Three panes, SSE events, endpoint list |
| [14–16. Delivery](PLAN.md#14-milestones) | Milestones, evaluation, risks |
| [17. Open decisions](PLAN.md#17-open-decisions-for-you) | What still needs your input |
| [Appendices](PLAN.md#appendix-a--setup-on-this-machine-windows-11-powershell) | Setup, bake-off script, config, layout, definition of done, day-one checklist |

---

## Still open

1. **CPU channel + exact model** — `Get-CimInstance Win32_PhysicalMemory` (one row = single-channel)
   and `Get-CimInstance Win32_Processor`. Changes the numbers, not the design.
2. **Your documents** — mostly digital PDFs, or scanned/office-heavy? Any non-English sources?
3. **Build or adopt** — this lean core, or stand up Open Notebook first and fork later?
4. **UI stack** — no-build htmx (leanest) vs. React (nicer to extend)?
5. **Audio overview** — must-have for v1, or is text + notes enough?
6. **Which sync provider** your vault uses (only affects placeholder handling and the first-export
   notice).

→ [PLAN.md §17](PLAN.md#17-open-decisions-for-you)

---

## Licensing notes

The app is intended to be permissively licensed. Third-party components worth checking before
distribution: model weights (`qwen3` is Apache-2.0; check Gemma and Llama terms), `nomic-embed-text`
(Apache-2.0), PyMuPDF (AGPL unless commercially licensed — verify before shipping), and
[OmniRoute](https://github.com/cheaper-inference/OmniRoute) (MIT, but the free provider tiers behind
it have their own terms).
