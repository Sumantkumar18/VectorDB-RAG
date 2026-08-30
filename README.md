# VectorDB — A Vector Database Built From Scratch in C++

I built this to understand how vector databases like Pinecone, Weaviate, and Chroma actually work under the hood, instead of just calling their APIs. It's a fully working vector database with a web UI, implementing three different nearest-neighbor search algorithms side by side, plus a complete RAG (Retrieval-Augmented Generation) pipeline powered by a local LLM.

No cloud dependencies, no external vector DB service — everything runs locally, including the language model.

## What it does

| Feature | Description |
|---|---|
| 3 Search Algorithms | HNSW (the production-grade approach), KD-Tree, and Brute Force — run all three side by side and compare |
| 3 Distance Metrics | Cosine similarity, Euclidean distance, Manhattan distance |
| 16D Demo Vectors | 20 pre-loaded semantic vectors across 4 categories (CS, Math, Food, Sports) |
| 2D PCA Scatter Plot | Live visualization of the semantic space — you can watch clusters form |
| Real Document Embedding | Paste in any text and Ollama embeds it with `nomic-embed-text` (768D) |
| RAG Pipeline | Ask a question about your documents → HNSW retrieves the relevant context → a local LLM answers |
| Full REST API | Insert, delete, search, benchmark, and inspect the HNSW graph structure directly |

## How the RAG pipeline works

```
Your Text
    │
    ▼
Ollama (nomic-embed-text)     ← turns text into a 768-dimensional vector
    │
    ▼
HNSW Index (C++)              ← indexes the vector in a multilayer graph
    │
    ▼
Semantic Search                ← finds nearest neighbors in vector space
    │
    ▼
Ollama (llama3.2)              ← reads the retrieved chunks, generates an answer
    │
    ▼
Answer
```

HNSW (Hierarchical Navigable Small World) is the same algorithm behind Pinecone, Weaviate, Chroma, and Milvus. It builds a multilayer graph where each layer above the base is progressively sparser — search starts at the top layer and zooms in, which is what gets you O(log N) complexity instead of the O(N) you'd get scanning every vector with brute force.

## Setup (Windows)

You'll need three things installed:

1. **MSYS2** — gives you a `g++` compiler
2. **Git**
3. **Ollama** — runs the local models

### 1. Install MSYS2 and g++

- Download the installer from [msys2.org](https://www.msys2.org) (or grab it directly from the [msys2-installer GitHub releases](https://github.com/msys2/msys2-installer/releases/latest) if the main site is slow to respond for you)
- Keep the default install path: `C:\msys64`
- Open **MSYS2 UCRT64** from the Start Menu (the orange icon — there are a few similarly named MSYS2 terminals, make sure it's UCRT64 specifically)
- Run:
  ```
  pacman -Syu
  ```
  (if it asks you to close and reopen the terminal to finish, do that and run it again)
- Then install the compiler:
  ```
  pacman -S mingw-w64-ucrt-x86_64-gcc
  ```
- Add `C:\msys64\ucrt64\bin` to your Windows PATH (Environment Variables → Path → New)
- Open a **new** terminal window and confirm:
  ```
  g++ --version
  ```

### 2. Install Git

Download from [git-scm.com](https://git-scm.com/download/win), install with defaults, verify with `git --version`.

### 3. Install Ollama and pull the models

Download from [ollama.com](https://ollama.com), then:

```
ollama pull nomic-embed-text
ollama pull llama3.2
```

Confirm both are there with `ollama list`. Budget ~8GB RAM for comfortable use — the two models together use about 3GB.

### 4. Clone and build

```
git clone https://github.com/Sumantkumar18/VectorDB-RAG.git
cd VectorDB-RAG
g++ -std=c++17 -O2 main.cpp -o db -lws2_32
```

Compiling takes 10–20 seconds and produces `db.exe`.

### 5. Run it

```
./db
```

You should see:

```
=== VectorDB Engine ===
http://localhost:8080
20 demo vectors | 16 dims | HNSW+KD-Tree+BruteForce
Ollama: ONLINE
  embed model: nomic-embed-text  gen model: llama3.2
```

Open `http://localhost:8080` in your browser.

## Using it

**Search tab** — type a concept (`binary tree`, `sushi`, `basketball`), pick an algorithm and distance metric, hit search. "Compare All Algos" runs all three at once so you can see the speed difference directly. The scatter plot shows all 20 vectors projected to 2D via PCA — the four categories visibly cluster, which is a nice way to *see* what "semantic similarity" actually means rather than just take it on faith.

**Documents tab** — paste real text (notes, an article, whatever). It gets chunked into overlapping 250-word pieces, each one gets a real 768D embedding from Ollama, and each is indexed separately.

**Ask AI tab** — once you've added some documents, ask a question about them. The question gets embedded, HNSW finds the 3 most relevant chunks, and `llama3.2` generates an answer grounded in just those chunks — you can click the context chips to see exactly what it used.

## REST API

| Method | Endpoint | Description |
|---|---|---|
| GET | `/search?v=...&k=5&metric=cosine&algo=hnsw` | K-NN search |
| POST | `/insert` | Insert a demo vector |
| DELETE | `/delete/:id` | Delete by ID |
| GET | `/items` | List all demo vectors |
| GET | `/benchmark?v=...&k=5&metric=cosine` | Compare all 3 algorithms |
| GET | `/hnsw-info` | Inspect the HNSW graph structure |
| POST | `/doc/insert` | `{"title":"...","text":"..."}` — embed and store |
| POST | `/doc/ask` | `{"question":"...","k":3}` — RAG retrieve + generate |
| GET | `/status` | Ollama status and model info |

## Why HNSW wins at high dimensions

KD-Tree pruning works by ruling out entire subtrees when they can't possibly contain a closer point — but that relies on axis-aligned distance bounds, and in high-dimensional space almost every point ends up near the boundary of the search hypersphere. Nothing gets pruned, and you're basically back to brute force. HNSW's graph-based approach sidesteps this entirely, which is why it holds up at 768 dimensions while KD-Tree degrades badly past about 20.

## Project structure

```
VectorDB-RAG/
├── main.cpp        ← C++ backend: HNSW, KD-Tree, Brute Force, REST API, RAG
├── httplib.h       ← single-header HTTP server library (cpp-httplib)
├── index.html      ← frontend: PCA scatter plot, chat UI, benchmarks
└── README.md
```

## Troubleshooting

| Problem | Fix |
|---|---|
| `Ollama: OFFLINE` in the header | Run `ollama serve` in a terminal |
| Embedding takes forever | Ollama is downloading the model on first use — wait a couple minutes |
| `g++: command not found` | `C:\msys64\ucrt64\bin` isn't on your PATH — add it and open a fresh terminal |
| Port 8080 in use | `netstat -ano \| findstr 8080` then `taskkill /PID <pid> /F` |
| LLM answers are slow | Normal on a laptop CPU (10–30s). Switch to `llama3.2:1b` for speed — `ollama pull llama3.2:1b`, then update `genModel` in `main.cpp` and recompile |

## Notes

This started from an open-source MIT-licensed base project and I've been building on it to go deeper on how vector search and RAG actually work internally, rather than treating them as black boxes.

## License

MIT — use it however you'd like.
