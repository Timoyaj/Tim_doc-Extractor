# Tim_doc-Extractor → **Tim Notebook**

A CPU-first, fully local alternative to Google's NotebookLM: add your documents, chat with
them, and get answers with clickable citations — powered by a local model through
[Ollama](https://ollama.com), on a computer **without a GPU**.

> **Status:** planning. No application code yet.
> Read **[PLAN.md](PLAN.md)** for the full design: architecture, data model, ingestion and
> retrieval pipeline, CPU performance tuning, milestones and evaluation.

## The short version

- One Python process, one SQLite file, one browser tab. No Docker, no Postgres, no cloud.
- Tuned for the target box: **12 GB RAM, 512 GB SSD, no GPU**. Chat runs on `qwen3:4b`
  ("snappy"), background artifacts on `qwen3:8b` ("quality"), embeddings on
  `nomic-embed-text`. See [PLAN.md §2.1](PLAN.md#21-hardware-profile-12-gb-ram--512-gb-ssd).
- Hybrid keyword + vector retrieval, strict citation checking, and a single-inference gate so
  the CPU is never split between jobs.
- Honest UX for 30–60 second answers: stage indicators, token streaming, a working stop button.

## Quick start (once v1 exists)

```bash
ollama pull qwen3:4b            # interactive chat (~2.6 GB)
ollama pull nomic-embed-text    # embeddings (~275 MB)
ollama pull qwen3:8b            # optional: quality tier for digests/artifacts (~5.2 GB)
python -m timnb                 # http://127.0.0.1:8765
```

Expect ~20 s to first token and ~40 s for a full answer on the snappy tier; the 8B tier is
roughly 2× slower and is used only for work you are not waiting on.

## Roadmap

| Milestone | Scope |
|---|---|
| M0 | Model bake-off on your actual hardware (12 GB profile) |
| M1 | Vertical slice: upload → chunk → retrieve → grounded answer |
| M2 | Usable app: queue, UI, streaming, citation drawer, notes |
| M3 | Digests and studio artifacts (FAQ, study guide, timeline, mind map) |
| M4 | Evaluation harness, OCR, packaging |
| M5 | Audio overview via local TTS (optional) |

See [PLAN.md §15](PLAN.md#15-open-decisions-for-you) for the open decisions that shape the
defaults — hardware, document mix, build-vs-adopt, and UI stack.
