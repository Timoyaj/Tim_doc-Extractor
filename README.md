# Tim_doc-Extractor → **Tim Notebook**

A CPU-first, fully local alternative to Google's NotebookLM: add your documents, chat with
them, and get answers with clickable citations — powered by a local model through
[Ollama](https://ollama.com), on a computer **without a GPU**.

> **Status:** planning. No application code yet.
> Read **[PLAN.md](PLAN.md)** for the full design: architecture, data model, ingestion and
> retrieval pipeline, CPU performance tuning, milestones and evaluation.

## The short version

- One Python process, one SQLite file, one browser tab. No Docker, no Postgres, no cloud.
- Chat model defaults sized for 8 GB RAM (`qwen3:4b` / `phi4-mini:3.8b`);
  `nomic-embed-text` for embeddings.
- Hybrid keyword + vector retrieval, strict citation checking, and a single-inference gate so
  the CPU is never split between jobs.
- Honest UX for 30–60 second answers: stage indicators, token streaming, a working stop button.

## Quick start (once v1 exists)

```bash
ollama pull qwen3:4b
ollama pull nomic-embed-text
python -m timnb          # http://127.0.0.1:8765
```

## Roadmap

| Milestone | Scope |
|---|---|
| M0 | Model bake-off on your actual hardware |
| M1 | Vertical slice: upload → chunk → retrieve → grounded answer |
| M2 | Usable app: queue, UI, streaming, citation drawer, notes |
| M3 | Digests and studio artifacts (FAQ, study guide, timeline, mind map) |
| M4 | Evaluation harness, OCR, packaging |
| M5 | Audio overview via local TTS (optional) |

See [PLAN.md §15](PLAN.md#15-open-decisions-for-you) for the open decisions that shape the
defaults — hardware, document mix, build-vs-adopt, and UI stack.
