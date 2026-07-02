# Local Intelligence Reference

Cheap, local-first ways to add smarts without a frontier API — and where the
cheapest one that works is *not* a model at all. Three layers, cheapest first:

- **Deterministic primitives** (hashing, LSH) — no model, exact or near-exact, ~free.
- **Task models** (embed, classify, rerank, extract, transcribe) — sub-1B,
  milliseconds on CPU.
- **Tiny generative LLMs** (0.5–1.5B) — run without a GPU.

This is a **concepts + reference** doc: what each thing *is*, what it's good for,
and the runtimes and models that serve it. It intentionally carries no
project-specific proposals — just the learning.

> **The through-line:** reach for the cheapest layer that solves the problem —
> a hash before an embedding, an embedding before a reranker, a task model
> before a generative one.

> **Fast-moving space.** Benchmarks, model ids, and operator coverage shift
> release-to-release. Treat the tables as a current map; verify the runtime's
> docs before pinning a version.

---

## Index

1. [Concepts in plain terms](#1-concepts-in-plain-terms) — what each job actually means
2. [Embeddings and hashing](#2-embeddings-and-hashing) — the two document primitives, and how they compose
3. [Runtimes](#3-runtimes) — how the landscape sorts out, plus the two tables
4. [Small models by job](#4-small-models-by-job) — the catalog that does the work
5. [Model type to runtime](#5-model-type-to-runtime) — quick lookup

---

## 1. Concepts in plain terms

The jobs a small model can do, in plain language. The catalog in §4 sorts models
by these.

- **Embedding** — turn text into a list of numbers (a vector) so similar
  meanings land near each other. This is what powers "find similar" / semantic
  search: embed everything once, then a query's vector is *close* to the
  vectors of relevant items. The core primitive under search, dedup, clustering.
  (Contrasted with hashing in §2.)

- **Transformer vs static embedding** — the normal kind (MiniLM, bge) runs a
  model on every query, so a word's vector shifts with context ("bank" differs
  in *river bank* vs *bank account*). Accurate, but a model pass and a ~90 MB
  file to ship. **Static** embeddings (Model2Vec / potion) instead precompute
  **one fixed vector per word into a lookup table**, once — at runtime there's no
  model, just look words up and average. Nanoseconds, tiny, single artifact; but
  the vector can't change with context, so quality is lower. It's the embedding
  version of choosing a lookup over a computation — reach for it when speed and
  "one file, no runtime" beat precision (rough dedup, first-pass filtering).

- **Reranker (cross-encoder)** — a second, sharper pass over search results.
  Retrieval gives you ~50 candidates ranked by embedding-distance; a reranker
  reads *query and candidate together* and re-scores each, pushing the truly
  relevant ones to the top. It's slower — one model pass **per candidate** — so
  you only run it on the shortlist, not the whole corpus. Big precision jump,
  paid per query. Its value scales **inversely with context budget**: with a
  huge context you can just include all candidates; with a tiny one, *which* few
  you keep is the whole game.

- **Classifier** — put text into a bucket: language ID, spam/not-spam,
  sentiment, intent, toxicity. Answers "which label," not "extract what."

- **NER & token tagging (extract entities)** — point at the spans in a sentence
  that are *things of a type* and label them. "Book a table for Sarah at Nando's
  on Friday" → `Sarah=PERSON, Nando's=ORG, Friday=DATE`. It doesn't understand
  the sentence — it **finds the useful nouns and labels them**, turning free text
  into structured fields. Fixed-label tools (spaCy) know a preset list;
  open/zero-shot tools (**GLiNER**) let you name the labels at runtime ("extract:
  invoice number, train number") with no retraining. Use it only for entities a
  regex can't pin down (names, places, fuzzy dates) — for rigid shapes (keys,
  UUIDs, prices) regex is cheaper and exact.

- **Segmentation** — image work: draw the exact pixel outline of an object, not
  just a box around it ("these pixels are *the cup*"). **SAM** is Meta's big
  model; **MobileSAM / FastSAM** are the shrunk versions that run on CPU/phone.
  Used for cut-outs, background removal, tap-to-select. Note: if you already
  have a text snapshot of a UI (accessibility tree / DOM), pixel vision is
  usually redundant.

- **Generative SLM** — a tiny LLM (0.5–1.5B) that *writes* text: route, extract
  to JSON, draft, classify-with-explanation. Won't reason like a frontier model,
  but runs locally with no GPU and no API round-trip. **GGUF** is its file
  format; **quantization** is shrinking the model's numbers (e.g. 16-bit → 4-bit)
  so it fits in RAM and runs fast, at a small quality cost.

---

## 2. Embeddings and hashing

The two document primitives. Both turn a document into a compact fixed-size
value, so they're easy to conflate — but they're near-opposites, and knowing
which one a problem wants is half the battle.

### Are they the same? No.

| | **Hashing** | **Embedding** |
|---|---|---|
| Goal | Detect **identity** — "same bytes?" | Detect **similarity** — "same meaning?" |
| Tiny input change | Completely different output (avalanche) | Almost identical output |
| Output | Opaque digest (SHA-256, blake3) | Vector of floats in a meaning-space |
| Compare by | Equality (`==`) | Distance / cosine similarity |
| Structure to exploit | None | Neighbors are meaningful — cluster, arithmetic |
| Cost | ~Free, deterministic | A model pass (a task model, §4) |

Hashing answers **"has this changed?"** Embedding answers **"what is this
like?"** A one-word edit flips a hash but barely moves an embedding — that
opposition *is* the point of each.

**The bridge — locality-sensitive hashing (LSH):** SimHash, MinHash. "Hashes"
deliberately built so *similar* inputs collide. Effectively cheap approximate
embeddings; the reason the two concepts feel adjacent. Use them to shrink a
candidate set before exact work, or to catch near-duplicates without a model.

### What can be done with each

**Hashing — cheap, exact, deterministic:**
- **Change detection / caching** — skip re-processing (or re-embedding) a doc
  whose content hash is unchanged. The big pipeline cost saver.
- **Dedup** — drop byte-identical docs before indexing.
- **Content-addressed storage** — store/reference a doc *by* its hash (what git
  itself does).
- **Integrity / tamper detection** — verify a doc wasn't modified.
- **Near-dup detection** (MinHash/SimHash) — find *almost*-identical docs
  (copy-paste, boilerplate) with no embedding model.

**Embeddings — semantic, fuzzy, model-driven:**
- **Semantic search / RAG retrieval** — the obvious one.
- **Clustering & topic discovery** — group unlabeled docs.
- **Classification / routing** — embed + a lightweight classifier over a rules engine.
- **Dedup by *meaning*** — catch two docs that say the same thing in different
  words (hashing can't).
- **Anomaly / drift detection** — flag a doc far from every known cluster.
- **Recommendation / "related docs"** — nearest neighbors.
- **Cross-lingual matching** — same meaning across languages (multilingual embedders).

### How they compose

Not competitors — different stages. Hashing is the **exact / identity / cheap**
layer; embeddings are the **semantic / similarity / expensive** layer. A good
doc pipeline uses hashing to avoid paying for embeddings it doesn't need:

1. **Hash first, as a gate** — content hash decides "new or changed?" If not,
   skip everything downstream.
2. **Embed second** — only for docs that passed the gate.
3. **Cache the embedding keyed by the hash** — identical content never gets
   re-embedded (embeddings cost a model pass; hashes are ~free).
4. **LSH-bucket to scale** — narrow candidates with a cheap similarity hash
   before exact cosine similarity.

The general lesson: the cheap deterministic primitive (hash) guards the door so
the expensive model (embedding) only runs where it earns its keep.

---

## 3. Runtimes

### How the landscape sorts out

Two runtimes cover almost everything on a single-box, local-first, JavaScript
stack:

- **`transformers.js`** — task models (embed / classify / rerank / NER / STT).
  ONNX Runtime as WASM + native Node bindings, runs in-process, `npm install`
  and done. (`fastembed-js` if you only want embeddings; `onnxruntime-node` for
  lower-level control.)
- **`llama.cpp` / GGUF** — generative SLMs, via `node-llama-cpp` in-process. It
  also produces embeddings, if you'd rather run a single runtime.

Everything else in the tables below falls into buckets you can usually skip on a
local box: **hardware pins** (OpenVINO, Core ML, ncnn — only for specific
silicon), **GPU-scaling servers** (vLLM, SGLang, TGI — for serving many
concurrent users), and **Rust runtimes** (`tract`, `Candle`, `Burn`) — excellent
but they graft a Rust toolchain onto a Node stack.

### 3a. Generative LLM runtimes

| Runtime | Format | Hardware | Best for | Zero-dep fit |
|---|---|---|---|---|
| **llama.cpp** | GGUF | CPU-first, GPU opt. | The foundation; local LLM anywhere | High (C++, single binary) |
| **Ollama** | GGUF | CPU/GPU | Easiest local UX; daemon + OpenAI API | Medium (wraps llama.cpp) |
| **llamafile** | GGUF | CPU/GPU | Ship a model as one executable | Very high |
| **node-llama-cpp** | GGUF | CPU/GPU | llama.cpp *in-process in Node* | High (in-process Node) |
| vLLM | safetensors | GPU req. | High-throughput serving, many users | Low (Python, heavy) |
| SGLang | safetensors | GPU req. | High-throughput + structured output | Low |
| TGI | safetensors | GPU req. | HuggingFace production serving | Low |
| MLX / mlx-lm | MLX | Apple Silicon | Fast local LLM on M-series | Medium |
| ExLlamaV2 | EXL2 | NVIDIA | Fastest single-GPU quantized | Low |
| TensorRT-LLM | TRT engine | NVIDIA | Max NVIDIA perf; complex | Low |
| Candle | safetensors/own | CPU/GPU | Rust-native small LLMs | High (Rust — needs a Rust toolchain) |
| CTranslate2 | CT2 | CPU/GPU | Fast translation + Whisper | Medium |

### 3b. Task-model runtimes (ONNX class — classifiers, embedders, vision, audio)

| Runtime | Format | Hardware | Best for | Zero-dep fit |
|---|---|---|---|---|
| **transformers.js** | ONNX | CPU/GPU (WASM+Node) | Embed/classify/rerank/NER/STT *in Node* | High (in-process Node) |
| ONNX Runtime | ONNX | CPU/GPU/NPU | Cross-framework reference runtime | Medium |
| ORT Web | ONNX | Browser (WASM) | In-browser inference, no backend | Medium |
| fastembed-js | ONNX | CPU | Embeddings-only, thin | High (Node) |
| tract | ONNX/TF | CPU | Pure-Rust, tiny, embeddable | Very high (Rust) |
| Burn | ONNX import | CPU/GPU/wgpu | Rust-native framework | High (Rust) |
| OpenVINO | ONNX/IR | Intel CPU/iGPU/NPU | Fastest on Intel hardware | Low |
| TFLite / LiteRT | TFLite | Mobile/edge | Android & embedded on-device | Medium |
| ExecuTorch | PTE | Mobile/edge | PyTorch-native on-device | Medium |
| Core ML | mlpackage | Apple | iOS/macOS Neural Engine | Medium (Apple-only) |
| ncnn / MNN | own | Mobile | Tiny fast mobile vision | High (C++, small) |
| GGML | GGUF | CPU/GPU | Tensor lib under llama.cpp | High |

---

## 4. Small models by job

What actually does the work. "Small" ≈ under ~1B params, often far under;
milliseconds on CPU, MB to a few hundred MB, one job well.

### Embeddings (text → vector, for search/similarity)
- **all-MiniLM-L6-v2** (~22M, ~90MB) — the default small embedder.
- **bge-small** (~33M) / **bge-base** — stronger retrieval, still small.
- **gte-small** — competitive alternative.
- **nomic-embed-text** — local-first favorite, 8k context, Apache.
- **Model2Vec / potion** — static embeddings (lookups, no transformer at
  inference). Absurdly fast, lower quality. Fits a zero-dep, single-artifact goal.
- *Use for:* semantic search, RAG retrieval, dedup, clustering, "find similar."

### Rerankers (reorder retrieved results by true relevance)
- **bge-reranker-base / -v2-m3**, **mxbai-rerank** — take query + candidates,
  score each. Retrieve 50, rerank to best 5. Big precision jump for little cost.
- *Cost note:* one model pass **per candidate, per query** — cheap next to an
  SLM's own inference, not free next to a fast API call.

### Classifiers (label content)
- **fastText** (Meta) — language ID + text classification, blazing fast, tiny.
- **DistilBERT / TinyBERT / MiniLM fine-tunes** — sentiment, topic, intent,
  spam, toxicity. Fine-tune on your labels → a few hundred MB or less.
- *Use for:* routing, tagging, moderation, triage.

### NER & token tagging (extract entities)
- **spaCy small** (`en_core_web_sm`, ~12MB) — names, orgs, locations, dates.
- **GLiNER** — extract **arbitrary** entity types named *at runtime*
  ("extract: invoice number, due date"), no fine-tuning. Small, genuinely
  useful for structured extraction from free text.

### Speech-to-text
- **Whisper tiny / base** (~39M / ~74M) via **whisper.cpp** — on-device
  transcription. base = sweet spot.
- **Moonshine, Distil-Whisper** — faster edge/streaming alternatives.

### Vision (small, on-device)
- **YOLO-nano / YOLOv8n** — object detection, real-time CPU.
- **MobileNet, EfficientNet-lite** — image classification.
- **MediaPipe** — face/hand/pose landmarks.
- **MobileSAM / FastSAM** — lightweight segmentation.

### Tiny generative LLMs (run locally, no GPU)
- **Qwen2.5 0.5B / 1.5B** — strong for size, good structured output / tool-call
  formatting.
- **SmolLM2** (135M / 360M / 1.7B) — HF small line, Apache.
- **Gemma 3 270M / 1B, Llama 3.2 1B** — instruction-tuned tiny.
- **TinyLlama 1.1B** — the long-running community small model.
- *Use for:* simple route/extract/classify, draft generation, on-device agents
  where 7B is too heavy. Won't reason like a frontier model; fine for "bucket
  this into one of 5" or "extract these fields as JSON."

### OCR & document
- **PaddleOCR (mobile), Tesseract, docTR** — text from images/scans.
- **Surya** — modern OCR + layout, multilingual.

---

## 5. Model type to runtime

Quick lookup — which runtime typically serves each model type in Node.

| Model type | Typical runtime (Node) |
|---|---|
| Text classifier | transformers.js |
| Embedding model | transformers.js / node-llama-cpp / fastembed-js |
| Reranker (cross-encoder) | transformers.js |
| NER / extraction (GLiNER) | transformers.js |
| Generative SLM | llama.cpp / node-llama-cpp |
| Speech-to-text | whisper.cpp |
| Vision (detection/pose) | ncnn / MNN / TFLite (mobile only) |
