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

- **k-Nearest Neighbors (kNN)** — the algorithm that turns embeddings into an
  answer: to decide about a new item, find the *k* closest stored items and let
  them **vote** (for a label) or **average** (for a value). No training and no
  parameters — it's just a **stored pile + a distance measure**, so you can add
  or drop examples instantly. Brute-force cosine for a few thousand vectors; a
  vector index (hnswlib, sqlite-vec, FAISS) for millions in milliseconds. It's
  the engine actually doing the work under semantic search, RAG retrieval,
  recommendation ("nearest items"), dedup, and the plain "kNN classifier." The
  catch that governs *all* of it: **kNN ranks on *similarity*, not *correctness*** —
  the nearest neighbor can be a confident wrong twin. It **proposes** the
  most-similar candidates; something that can *fail* (a rule, a test, a reranker,
  a human) has to **dispose**. Similarity is a candidate generator, never a truth
  signal.

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

- **Retrieval-augmented memory (parametric vs non-parametric)** — a model keeps
  knowledge in two places. **Parametric** memory is baked into the weights at
  training time: capacious but frozen, expensive to update, and blurry on rare
  "long-tail" facts. **Non-parametric** memory is a datastore you **kNN over at
  run time**: add, edit, or delete a fact instantly, and it's sharp on exactly
  the long-tail the weights forget. **RAG** is this at the prompt level (retrieve
  passages, paste into context); **kNN-LM** and **RETRO** push it deeper —
  blending a nearest-neighbor vote from the datastore into the model's own
  next-token distribution (a normal LM with a lookup bolted on, not a new kind of
  model). Why it matters on a local box: a tiny SLM plus a local vector store can
  recall facts it never had the capacity to memorize — **offload memory to a
  lookup instead of buying a bigger model**, the same cheapest-layer move as
  everything else here.

- **Matching in embedding space — one operation, three hats** — "compare vectors
  for closeness" shows up in three places, which is why retrieval, model
  internals, and newer training tricks all feel adjacent. (1) **Retrieval** — kNN
  over a datastore (non-parametric, above). (2) **A generative model's own output
  head** — the final hidden state is scored against every token's embedding
  (`logits = hidden · Eᵀ`) and the closest wins, so next-token prediction is
  itself a giant nearest-match over the vocabulary (parametric). (3)
  **Predict-in-embedding-space training (JEPA)** — instead of reconstructing raw
  pixels/tokens, predict the *embedding* of the masked part and match it, forcing
  the model to represent only the abstract, predictable structure. JEPA is mature
  for vision/video; the text version (predicting sentence embeddings — "concept
  models") is still frontier and immature, so for local work the embedding-match
  tools that actually pay off today are **kNN retrieval and the model's own
  head**, not latent-space text prediction.

- **Featural vs relational similarity (the analogy gap)** — embeddings encode
  **featural** similarity: things *described in similar words* land nearby.
  Analogy is **relational** similarity: same *structure*, different surface —
  "project management applies to cooking" (dependencies, sequencing, buffers,
  critical path) shares no vocabulary, so no cosine will ever surface it. This
  is a ceiling better embeddings can't fix (Gentner's *structure-mapping*, from
  the 1980s, is still the cleanest statement of it). Two honest ways past it,
  and neither is a smarter vector: (1) **relatedness from use** — if two items
  were ever retrieved into the *same task*, they're related regardless of
  embedding distance; log co-retrieval and learn association edges from it, so
  a cross-domain bridge becomes retrievable as a fact of use once any real
  session crosses it; (2) **truth from consequences** — similarity *proposes*
  candidates, something that can fail (a test, a rule, a human) *disposes*;
  never let ranking be the correctness discriminator. (Extends the kNN caveat
  above from "wrong twin" to "invisible cousin.")

- **Spotlight vs floodlight (why models have no "background thinking")** —
  transformer attention is literally **spotlight**: softmax is a concentration
  operator; the architecture's whole job is to narrow onto a query. There is no
  ambient process — no idle time, no diffuse simmering, nothing like human
  *incubation* (low-intensity activation spreading over hours, plus offline
  **consolidation** — sleep replay recombining the day's traces and
  strengthening cross-links). The mechanical floodlight analogues all live
  *outside* the model, and all are cheap: (1) **scheduled consolidation** — a
  query-free offline pass over accumulated traces looking for what recurs
  (floodlight by schedule, not architecture); (2) **spreading activation over
  an associative graph** — activation leaks outward from a seed with decay,
  instead of ranking against a query (the classic computational model of
  diffuse attention; query-driven top-k is the spotlight, decay-bounded
  spreading is the floodlight); (3) **idle-time "dreaming"** — walk the store
  without any query, proposing candidate cross-domain edges, stamped
  *unverified* until a later task uses or falsifies them (not answering, just
  noticing). Floodlight-in-the-weights needs a persistent-state architecture —
  a research bet; floodlight-as-scheduled-process is buildable today on access
  logs.

- **Hebbian association (relatedness earned by use, not inferred from content)** —
  Hebb's rule ("fire together, wire together") never inspects what two items
  *are* — only whether they were **active at the same time**. That inverts the
  embedding approach: content-comparison relates items by description (and so
  can never cross a vocabulary gap); a Hebbian edge relates them by observed
  **co-use** — the first time one real task retrieves both, the bridge exists
  and is retrievable from then on. The update is **local and ~free**: a decayed
  pair-counter over retrieval logs *is* a Hebbian learner (below even the
  embedding on the cheapest-layer ladder — it's just counting). The engineering
  value is as much in the known **pathologies and their classic fixes** as in
  the rule: pure Hebbian only strengthens, so it runs away (rich-get-richer;
  everything wires to everything). The fixes all map to one-liners — (1)
  **decay**: edges weaken unless re-reinforced; ACT-R's base-level activation
  (availability as a function of frequency *and* recency of past use) is the
  ready-made, engineering-hardened formula, and it's the piece most systems
  implement first; (2) **bounded/normalized weights** (Oja's rule): cap a
  node's total edge mass / normalize by degree so hub items can't relate to
  everything; (3) **anti-Hebbian negative evidence**: co-activation followed
  by a *bad outcome* weakens the edge; (4) **order (STDP)**: A-then-B
  strengthens a *directed* edge — sequence knowledge, enabling prefetch. The
  discipline most deployments skip is (3), and it has two layers. First,
  **retrieved-together ≠ used-together**: co-*retrieval* is the ranker's
  behavior, not the task's — wire edges from it and the graph just re-learns
  the retriever's own closeness bias wearing an "earned from experience"
  costume (a feedback loop, worse than useless because it looks independent).
  Second, even genuine co-use in a *failed* task is no evidence. Neuroscience
  patched this exact hole with **three-factor learning rules + eligibility
  traces**: co-activation writes only a *tentative, decaying trace* — nothing
  durable — which converts to a real weight change only if a third factor (a
  reward signal; dopamine, biologically) arrives within the trace's window.
  Engineering translation: the edge condition is a conjunction — **surfaced ∧
  evidence-of-use ∧ validated outcome** — and each factor kills a distinct
  failure mode. Use-evidence, strongest to weakest: (a) the
  **failure-transition** signal — the item surfaced between a failed and a
  passing attempt (near-counterfactual attribution, often already sitting in
  the logs); (b) **artifact overlap** — the item's content is checkably
  reflected in what was produced; (c) **self-citation** — the agent says it
  used it (self-report: a candidate to verify against (b), never proof).
  Decay without this gate is the common halfway point; the full conjunction
  turns "retrieved together" into "**worked together**" — relatedness with a
  truth component. Composition: Hebbian graphs only know **paths already
  traveled** (no never-crossed bridge), so pair with embeddings — cosine
  proposes cold-start seeds (nearness), spreading activation over the earned
  graph expands them across domain boundaries (relatedness), and the outcome
  check disposes. Each layer does the one thing the others can't.

- **Stigmergy (coordination through the environment, not messages)** — ants
  don't message each other; they coordinate by **leaving traces in a shared
  environment** (pheromones) that other ants read and reinforce. That's
  stigmergy, and it's the actual mechanism behind most "swarm intelligence" in
  nature — agents never hold a protocol; the *environment* holds the state.
  The engineering translation: an append-only event log or a shared persistent
  store that agents read and write *is* a stigmergic medium — coordination
  falls out of trace-reading, with no inter-agent protocol to design, version,
  or break. Cheaper and more robust than message-passing choreography (a trace
  outlives the agent that wrote it; a message doesn't), and it composes with
  the Hebbian entry above: traces stamped with *outcomes* let later agents
  weight them by what worked, not just what was written.

- **Emergent coordination and where the arbiter goes (boids, flocks, swarms)** —
  a flock has no blueprint; each bird follows ~3 local rules (separate, align,
  cohere) and the flock **emerges from the relations**. The seductive
  engineering read is "stop hard-coding the pipeline; initialize capable agents
  under simple local rules and let the workflow self-organize." The part that
  read skips: **emergence is safe exactly where consequences are real,
  immediate, and unfakeable.** Flocks work because *physics is the arbiter* —
  no bird papers over a collision; markets work because P&L is
  consequence-bearing. Digital agents' consequences are deferred and fakeable,
  so a self-organizing system with no external ground truth is
  self-certification at scale — looking-good becomes the evolutionary
  attractor. The design law: **let the middle emerge; never let the arbiter
  emerge.** Plans, roles, topology, memory strategy — fine as runtime outputs;
  the verification seam (an executable check that can fail) must stand outside
  whatever self-organizes. Corollaries: cost caps and permission scopes double
  as the *selection pressure* that shapes emergent behavior (the
  constraint-designer role); one agent assembling its own workflow isn't
  emergence, just planning — real emergence needs **population + iteration +
  retained selection** (what worked must persist across runs, gated by
  outcomes per the Hebbian entry, or the population learns to look green
  rather than be green).

- **Plan-in-latent-space and the arbiter gap (JEPA/MuZero vs chess)** —
  "predict in a learned latent space, plan by rolling forward" is the old
  model-based-planning branch returning (MuZero already planned in a learned
  latent; straight MCTS ancestry). The crucial difference from its chess
  origins: chess had a **perfect simulator and a ground-truth terminal signal**
  (win/loss) — the tree search was honest because leaf evaluation was anchored
  to a rule of the game. A JEPA-style model must *learn* the simulator, and
  its signal is prediction error in its own representation space — still a
  closeness measure. Latent prediction error tells you **surprising, not
  false**. So the field is borrowing chess's *search* but cannot borrow chess's
  *arbiter* — which is why the practical scaffolding trend (sandboxes,
  reversible execution traces, checkable rewards) all sits on the arbiter side:
  since the internal signal can't be made truthful, make the *consequences*
  observable and ground on those. Anything you build locally should follow the
  same split: representation proposes, consequence disposes.

- **Pretraining vs fine-tuning (where the compute actually is)** — two very
  different costs hide under "train a model." **Pretraining** teaches a model
  language from scratch — thousands of GPU-hours, real money; this is the thing
  you *download*, never the thing you do. **Fine-tuning** rides on top of a
  pretrained model to specialize it on your labels — a free Colab T4 or a modest
  GPU, minutes to a couple of hours, even CPU-viable for small data. So "can I
  train my own X?" almost always means *fine-tune*, and the honest answer is
  usually yes, cheaply. Inference after is milliseconds on CPU either way. Reach
  for it only after the zero-training options (regex, zero-shot NER like GLiNER,
  an off-the-shelf classifier) fall short — the same cheapest-layer ladder as
  everything else here.

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

### PII detection & de-identification
Not a single model — a *composed pipeline*, and the cleanest instance of this
doc's whole thesis (§2's "how they compose"): cheap exact layer guards the door,
model layer runs only where it earns its keep.
- **Microsoft Presidio** (Apache-2.0, Python; `github.com/microsoft/presidio`) —
  an assembled PII framework, three parts: **Analyzer** (finds PII via regex
  pattern-recognizers + deny-lists + an NLP engine — spaCy/transformers — plus
  *context boosting*, e.g. a nearby "SSN:" raises confidence; returns typed spans
  with scores), **Anonymizer** (applies operators to those spans — replace,
  redact, mask, hash, encrypt), and **Image Redactor** (OCR → redact PII in
  images/scans, incl. DICOM). Extensible with custom recognizers. Python — on a
  Node stack, replicate the pattern (regex + GLiNER via transformers.js) rather
  than adopt it directly.
- **The layered stack it embodies:** regex for structured PII (email, SSN, card
  via Luhn, phone, IP) → **GLiNER** zero-shot for fuzzy PII (names, addresses,
  DOB) with no training → **fine-tune** a token classifier only if your domain
  needs it.
- **Data if you fine-tune:** `ai4privacy/pii-masking-200k` (HuggingFace) —
  labeled PII spans across many types and languages.
- *Use for:* redaction, de-identification, log/analytics scrubbing, safe-to-share
  exports.

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
| Vector index (kNN / retrieval) | hnswlib-node / sqlite-vec / faiss-node (brute-force cosine for small N) |
| Reranker (cross-encoder) | transformers.js |
| NER / extraction (GLiNER) | transformers.js |
| Generative SLM | llama.cpp / node-llama-cpp |
| Speech-to-text | whisper.cpp |
| Vision (detection/pose) | ncnn / MNN / TFLite (mobile only) |
