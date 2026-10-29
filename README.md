# Tim_doc-Extractor → **Tim Notebook**

A CPU-first, fully local alternative to Google's NotebookLM: add your documents, chat with
them, and get answers with clickable citations — powered by a local model through
[Ollama](https://ollama.com), on a computer **without a GPU**.

> **Status:** planning. No application code yet.
> Read **[PLAN.md](PLAN.md)** for the full design: architecture, data model, ingestion and
> retrieval pipeline, CPU performance tuning, milestones and evaluation.

## The short version

- One Python process, one SQLite file, one browser tab. No Docker, no Postgres, no cloud.
- Tuned for the actual target box: **Windows 11, 12 GB RAM, 512 GB SSD, 4-core 10th-gen
  Intel, no discrete GPU.** Two tiers instead of one: `qwen3:1.7b` for quick answers (~25 s),
  `qwen3:4b` for real questions and digests (~50 s), `nomic-embed-text` for embeddings. The
  8B model is deliberately *not* a default — see
  [PLAN.md §2.1–2.2](PLAN.md#21-hardware-profile-windows-11--12-gb--512-gb-ssd--4-core-10th-gen-intel).
- Hybrid keyword + vector retrieval, strict citation checking, and a single-inference gate so
  the CPU is never split between jobs.
- Honest UX for 30–60 second answers: stage indicators, token streaming, a working stop button.

## Quick start (once v1 exists)

```powershell
ollama pull nomic-embed-text    # embeddings (~275 MB)
ollama pull qwen3:1.7b          # quick answers (~1.4 GB)
ollama pull qwen3:4b            # real questions (~2.6 GB)

py -3.12 -m venv .venv
.venv\Scripts\activate
pip install fastapi uvicorn pymupdf4llm markitdown trafilatura sqlite-vec psutil httpx
python -m timnb                 # http://127.0.0.1:8765
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
| M4 | Evaluation harness, OCR, packaging |
| M5 | Audio overview via local TTS (optional) |

See [PLAN.md §15](PLAN.md#15-open-decisions-for-you) for the open decisions that shape the
defaults — hardware, document mix, build-vs-adopt, and UI stack.
