# Tim_doc-Extractor → **Tim Notebook**

A CPU-first, fully local alternative to Google's NotebookLM: add your documents, chat with
them, and get answers with clickable citations — powered by a local model through
[Ollama](https://ollama.com), on a computer **without a GPU**.

> **Status:** planning. No application code yet.
> Read **[PLAN.md](PLAN.md)** for the full design: architecture, data model, ingestion and
> retrieval pipeline, CPU performance tuning, milestones and evaluation.

## The short version

- One Python process, one SQLite file, one browser tab. No Docker, no Postgres, no cloud
  by default.
- Tuned for the actual target box: **Windows 11, 12 GB RAM, 512 GB SSD, 4-core 10th-gen
  Intel, no discrete GPU.** Two tiers instead of one: `qwen3:1.7b` for quick answers (~25 s),
  `qwen3:4b` for real questions and digests (~50 s), `nomic-embed-text` for embeddings. The
  8B model is deliberately *not* a default — see
  [PLAN.md §2.1–2.2](PLAN.md#21-hardware-profile-windows-11--12-gb--512-gb-ssd--4-core-10th-gen-intel).
- Hybrid keyword + vector retrieval, strict citation checking, and a single-inference gate so
  the CPU is never split between jobs.
- Honest UX for 30–60 second answers: stage indicators, token streaming, a working stop button.
- **Optional free cloud models** via [OmniRoute](https://github.com/cheaper-inference/OmniRoute),
  a local MIT-licensed gateway on `127.0.0.1:20128`. When the 1.7B/4B model isn't enough, one
  click answers the same question with a 70B-class free model in a few seconds. **Off by
  default**, per notebook, and every answer is labelled with who produced it — see
  [PLAN.md §10](PLAN.md#10-optional-online-mode-free-models-via-omniroute).
- **One click starts it.** No terminal, no copy-pasted commands: the app installs the gateway if
  needed, launches it bound to loopback, mints its own chat-scoped key and stops it again on
  disconnect ([PLAN.md §10.8](PLAN.md#108-one-click-connect-the-app-starts-the-gateway)).
- **Notes land in your Obsidian vault.** Any note or saved answer can be sent straight into a
  vault as plain Markdown — YAML frontmatter, working wikilinks to source notes, atomic writes,
  and re-export that never clobbers an edit you made in Obsidian. Vault auto-detected; no plugin,
  no Obsidian running required. See
  [PLAN.md §11](PLAN.md#11-obsidian-notes-land-in-your-vault).

## Local by default

The app is designed to work with the network cable unplugged. Documents, chunks, embeddings,
notes and the search index never leave the machine. Online mode is a **per-notebook policy**
(`local_only` by default), and even when enabled only the 3–6 retrieved chunks for a single
question are sent — never a whole document, never the index. Embeddings are always computed
locally. See [PLAN.md §10.1](PLAN.md#101-the-trade-stated-plainly) for the trade in full.

## Quick start (once v1 exists)

```powershell
ollama pull nomic-embed-text    # embeddings (~275 MB)
ollama pull qwen3:1.7b          # quick answers (~1.4 GB)
ollama pull qwen3:4b            # real questions (~2.6 GB)

py -3.12 -m venv .venv
.venv\Scripts\activate
pip install fastapi uvicorn pymupdf4llm markitdown trafilatura sqlite-vec psutil httpx
python -m timnb                 # http://127.0.0.1:8765

# optional: free cloud models when the local 4B isn't enough.
# No commands — in the app, Settings → Free models → Connect OmniRoute.
# optional: Settings → Obsidian → pick your vault; then "Send to Obsidian" on any note.
```

First read [PLAN.md §9.5](PLAN.md#95-windows-11-specifics-do-these-once-in-this-order) — four
Windows-specific steps (env vars, tray restart, Defender exclusions, keeping the database out
of OneDrive) that save hours. Tune the defaults with the
[day-one checklist](PLAN.md#appendix-f--day-one-checklist-for-this-laptop-45-minutes-before-writing-code).

## Roadmap

| Milestone | Scope |
|---|---|
| M0 | Day-one setup + bake-off on your actual laptop (Appendix F) |
| M1 | Vertical slice: upload → chunk → retrieve → grounded answer |
| M2 | Usable app: queue, UI, streaming, citation drawer, notes |
| M3 | Digests and studio artifacts (FAQ, study guide, timeline, mind map) |
| M4 | Optional free-cloud backend via OmniRoute (egress guard, attribution, audit) |
| M5 | Evaluation harness, OCR, packaging |
| M6 | Audio overview via local TTS (optional) |

See [PLAN.md §17](PLAN.md#17-open-decisions-for-you) for the open decisions that shape the
defaults — hardware, document mix, build-vs-adopt, and UI stack.
