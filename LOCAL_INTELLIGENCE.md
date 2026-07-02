# Local Intelligence Reference

Cheap, local-first ways to add smarts to a Node, zero-dep suite — and where the
cheapest one that works is *not* a model at all. Three layers matter here:
**deterministic primitives** (hashing, LSH — no model, exact or near-exact,
covered in §3.5), **task models** (embed, classify, rerank, extract, transcribe
— sub-1B, milliseconds on CPU), and **tiny generative LLMs** (0.5–1.5B, run
without a GPU). This doc maps the runtimes, the models, the primitives, what each
is good for, and where one actually belongs in a project.

> **The through-line:** reach for the cheapest layer that solves the problem —
> a hash before an embedding, an embedding before a reranker, a task model before
> a generative one. Every "yes" in §4–5 clears that bar; most ideas don't.

> **Fast-moving space.** Benchmarks, model ids, and operator coverage shift
> release-to-release. Treat the tables as a current map; verify the runtime's
> docs before pinning a version.

---

## 0. The two runtimes that cover a Node suite

You don't need the whole zoo. For a single-box, local-first, JavaScript stack,
two runtimes do everything:

- **`transformers.js`** — task models (embed / classify / rerank / NER / STT).
  ONNX Runtime as WASM + native Node bindings, runs in-process, `npm install`
  and done. (`fastembed-js` if you only want embeddings; `onnxruntime-node` for
  lower-level control.)
- **`llama.cpp` / GGUF** — generative SLMs, via `node-llama-cpp` in-process. It
  also produces embeddings, if you'd rather run a single runtime.

Everything else in §1–2 is here for completeness and falls into two buckets you
can skip on a local box: **hardware pins** (OpenVINO, Core ML, ncnn — only for
specific silicon) and **GPU-scaling servers** (vLLM, SGLang, TGI — for serving
many concurrent users). The Rust runtimes (`tract`, `Candle`, `Burn`) are
excellent but mean grafting a Rust toolchain onto a Node stack — noted where
they appear, not recommended here.

---

## 1. Generative LLM runtimes

| Runtime | Format | Hardware | Best for | Zero-dep fit |
|---|---|---|---|---|
| **llama.cpp** | GGUF | CPU-first, GPU opt. | The foundation; local LLM anywhere | High (C++, single binary) |
| **Ollama** | GGUF | CPU/GPU | Easiest local UX; daemon + OpenAI API | Medium (wraps llama.cpp) |
| **llamafile** | GGUF | CPU/GPU | Ship a model as one executable | Very high |
| **node-llama-cpp** | GGUF | CPU/GPU | llama.cpp *in-process in Node* | High — **generative pick** |
| vLLM | safetensors | GPU req. | High-throughput serving, many users | Low (Python, heavy) |
| SGLang | safetensors | GPU req. | High-throughput + structured output | Low |
| TGI | safetensors | GPU req. | HuggingFace production serving | Low |
| MLX / mlx-lm | MLX | Apple Silicon | Fast local LLM on M-series | Medium |
| ExLlamaV2 | EXL2 | NVIDIA | Fastest single-GPU quantized | Low |
| TensorRT-LLM | TRT engine | NVIDIA | Max NVIDIA perf; complex | Low |
| Candle | safetensors/own | CPU/GPU | Rust-native small LLMs | High (Rust — needs a Rust toolchain) |
| CTranslate2 | CT2 | CPU/GPU | Fast translation + Whisper | Medium |

## 2. Task-model runtimes (ONNX class — classifiers, embedders, vision, audio)

| Runtime | Format | Hardware | Best for | Zero-dep fit |
|---|---|---|---|---|
| **transformers.js** | ONNX | CPU/GPU (WASM+Node) | Embed/classify/rerank/NER/STT *in Node* | **Task-model pick** |
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

## 3. Small models by job (what actually does the work)

"Small" ≈ under ~1B params, often far under; milliseconds on CPU, MB to a few
hundred MB, one job well.

### Concepts in plain terms

The catalog below sorts models by job. What those jobs actually mean:

- **Embedding** — turn text into a list of numbers (a vector) so similar
  meanings land near each other. This is what powers "find similar" / semantic
  search: embed everything once, then a query's vector is *close* to the
  vectors of relevant items. The core primitive under search, dedup, clustering.

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
  paid per query.

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
  Used for cut-outs, background removal, tap-to-select. Listed for completeness
  — a text-snapshot (a11y/DOM) agent never needs pixels.

- **Generative SLM** — a tiny LLM (0.5–1.5B) that *writes* text: route, extract
  to JSON, draft, classify-with-explanation. Won't reason like a frontier model,
  but runs locally with no GPU and no API round-trip. **GGUF** is its file
  format; **quantization** is shrinking the model's numbers (e.g. 16-bit → 4-bit)
  so it fits in RAM and runs fast, at a small quality cost.

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
  useful for structured extraction. **Key model for the email + PII ideas (§5).**

### Speech-to-text
- **Whisper tiny / base** (~39M / ~74M) via **whisper.cpp** — on-device
  transcription. base = sweet spot.
- **Moonshine, Distil-Whisper** — faster edge/streaming alternatives.

### Vision (small, on-device)
- **YOLO-nano / YOLOv8n** — object detection, real-time CPU.
- **MobileNet, EfficientNet-lite** — image classification.
- **MediaPipe** — face/hand/pose landmarks.
- **MobileSAM / FastSAM** — lightweight segmentation.
- *Note:* barebrowse/baremobile prune **accessibility/DOM snapshots as text**
  by design — vision here would be **redundant**, not additive. Skip unless the
  a11y tree genuinely can't see something.

### Tiny generative LLMs (run locally, no GPU)
- **Qwen2.5 0.5B / 1.5B** — strong for size, good structured output / tool-call
  formatting.
- **SmolLM2** (135M / 360M / 1.7B) — HF small line, Apache.
- **Gemma 3 270M / 1B, Llama 3.2 1B** — instruction-tuned tiny.
- **TinyLlama 1.1B** — the long-running community small model.
- *Use for:* simple route/extract/classify, draft generation, on-device agents
  where 7B is too heavy. Won't reason like Claude; fine for "bucket this into
  one of 5" or "extract these fields as JSON."

### OCR & document
- **PaddleOCR (mobile), Tesseract, docTR** — text from images/scans.
- **Surya** — modern OCR + layout, multilingual.

### Model type → runtime (quick lookup)
| Model type | Typical runtime (Node) |
|---|---|
| Text classifier | transformers.js |
| Embedding model | transformers.js / node-llama-cpp / fastembed-js |
| Reranker (cross-encoder) | transformers.js |
| NER / extraction (GLiNER) | transformers.js |
| Generative SLM | llama.cpp / node-llama-cpp |
| Speech-to-text | whisper.cpp |
| Vision (detection/pose) | ncnn / MNN / TFLite (mobile only) |

---

## 3.5 Embeddings & hashing — the two document primitives

Both turn a document into a compact fixed-size value, so they're easy to
conflate. They're near-opposites. This is the pairing behind the doc-handling in
**gitdone**, and the same shape applies to **CSMA**.

### Are they the same? No.

| | **Hashing** | **Embedding** |
|---|---|---|
| Goal | Detect **identity** — "same bytes?" | Detect **similarity** — "same meaning?" |
| Tiny input change | Completely different output (avalanche) | Almost identical output |
| Output | Opaque digest (SHA-256, blake3) | Vector of floats in a meaning-space |
| Compare by | Equality (`==`) | Distance / cosine similarity |
| Structure to exploit | None | Neighbors are meaningful — cluster, arithmetic |
| Cost | ~Free, deterministic | A model pass (task-model, §3) |

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

### How they compose (the gitdone → CSMA pattern)

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

This is the §4/§6 discipline in miniature: the deterministic primitive (hash)
guards the door so the model (embedding) only runs where it earns its keep.

---

## 4. Where a model earns its place in the suite

Most of the suite is plumbing that calls a frontier model over an API and should
**stay model-free**. A local model earns its place in exactly these spots:

| Repo | Current mechanism | Model idea | Verdict |
|---|---|---|---|
| **litectx** | FTS5 keyword + **optional ONNX embeddings** (~23MB) + graph rank | add a **reranker** | Opt-in only, for the **SLM/tight-context** case — see §5. |
| **bareguard** | deterministic rules + **regex secret redaction** (keys/tokens) | **GLiNER PII detection** | Yes — but **advisory → ask-human only**, never auto-deny. See §5. |
| **coding-assistant** | already a llama.cpp/GGUF wrapper | pick which SLM | Already done; tables just say *which* (Qwen2.5-0.5B/1.5B, SmolLM2). |
| flightlog / pulselog | exact grouping by stable `where` field | semantic error clustering | **No.** The `where` convention already solved this deterministically; a model adds only nondeterminism. Only helps if grouping by raw *message* — which the design avoids. |
| barebrowse / baremobile | pruned a11y/DOM snapshots (text) | on-device vision | **No.** Redundant with the chosen text-snapshot approach. |

**Through-line:** each tool already replaces "ask a model" with a cheap
deterministic primitive (FTS5, rule allowlists, the `where` convention). The bar
for adding a model on top is therefore *high* — it must beat a mechanism that is
already fast, free, and predictable.

---

## 5. Concrete ideas worth building

### 5a. bareguard — PII-suspect → ask-human lane *(the one real feature)*
- **Gap:** regex catches *structured* secrets (keys, `Bearer …`) — it can't
  catch *unstructured* PII (names, addresses, free-text emails). No regex has a
  signature for those. GLiNER is the only thing that finds them.
- **Constraint:** bareguard's spine is **deterministic** ("structured rule
  primitives, not LLM judgments"). A probabilistic model that *sometimes* misses
  PII must **never** drive an automatic `deny` — that poisons the guarantee.
- **Design:** GLiNER (via transformers.js) runs as an **advisory signal** that
  routes a suspected match into the existing **`ask-human`** lane, clearly
  labelled probabilistic. Fits the ask/allow/deny gate without touching the
  deterministic floor.
- **Why it's worth it:** fills a true blind spot, in the one place no regex
  substitutes, without breaking the design.

### 5b. litectx — opt-in reranker (SLM-only)
- **Insight:** a reranker's value scales **inversely with context budget.**
  - Frontier model (1M ctx): stuff 50 loose memories, let it sort — reranker
    adds ~nothing.
  - **SLM (4–8k ctx):** you fit only **3–5 memory slots**; *which* few is the
    whole game, and a wrong pick can't be recovered in-context. Reranker decides
    the slots.
- **Latency worry dissolves here:** an SLM generating on CPU is the slow part;
  scoring ~20 candidates with a tiny reranker is noise beside it. (The latency
  only bit if the consumer were a fast API call — not litectx's audience.)
- **Turn it on when all three hold:** small context window **and** recall
  returns more close-scoring candidates than fit **and** acting on the wrong
  memory is costly (agent *does* something).
- **Skip when:** few memories total (include them all) or recall already has a
  clear winner.
- **Positioning line for docs:** *"Enable if you feed a small-context model and
  see the right memory land at rank 4 instead of rank 1."* Off by default, like
  embeddings already are.

### 5c. Standalone toys to play with
New playgrounds, not additions to existing repos. Ranked by fit + play-value.

1. **Email "pre-brain" — intent classifier + GLiNER extractor at the MTA.**
   *(strongest fit)* The email-native repos (latefyi, gitdone, addypin,
   mailproof) parse inbound mail with brittle regex today. A small classifier
   ("command? reply? junk?") + GLiNER ("pull: train number, date, destination")
   turns fragile parsing into robust structured commands — a cheap
   deterministic-ish pre-filter below the agent.

2. **Embedder shipped as one executable.** *(most on-brand)* llamafile-style
   single artifact: run it, it speaks HTTP, returns vectors. No Python, no model
   download for the consumer. The embedding substrate litectx (or anything)
   points at. Low utility ceiling, perfect taste-fit.

3. **Local voice-command for multis / beeperbox.** whisper.cpp tiny/base
   on-device: speak → transcribe locally → feed the agent. No cloud STT, no
   audio leaves the box. On-brand for the chat-agent direction.

4. **`presecret` — semantic secret + PII pre-commit hook.** *(de-risks 5a)*
   Regex blocks structured secrets outright; GLiNER *warns* on suspected PII
   (advisory, not deny — same discipline as bareguard). Immediately useful,
   tiny scope, and it's the GLiNER-in-Node spike you'd need for 5a anyway.
   **Build this first** — one toy de-risks the one real feature.

5. **Semantic dedup/cluster for JSONL.** *(standalone toy only)* The suite
   writes JSONL everywhere (flightlog, pulselog, litectx). A tool that clusters
   lines by meaning for eyeballing error variants / memory overlap. **Keep it a
   standalone inspection toy** — never wire it into the deterministic digest
   (see §4).

---

## 6. Anti-bloat checklist (read before adding any model)

1. **Is there already a deterministic primitive** doing this (FTS5, a rule, a
   convention like `where`)? If yes, a model probably adds nondeterminism, not
   value.
2. **Does the model *drive a decision* or just *advise*?** In a governance/floor
   context, probabilistic → advisory/`ask-human` only, never auto-deny.
3. **Is the value inverse to context budget** (rerank) or **filling a true blind
   spot** (unstructured PII)? Those are the two shapes worth it.
4. **Does it stay one artifact in Node** (transformers.js / node-llama-cpp)? If
   it needs a Rust toolchain or a Python venv, it's fighting the suite's ethos.
5. **Default off, opt-in, clearly labelled.** Match how litectx ships embeddings.

**Two runtimes, and really two features worth building: the bareguard PII lane
(5a) and — for SLM consumers only — the litectx reranker (5b). Everything else
is a toy or a no.**
