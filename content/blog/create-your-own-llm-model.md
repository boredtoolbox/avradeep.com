---
date: '2026-03-22T23:27:29+08:00'
draft: false
showtoc: true
tags: ["llm", "rag", "ai", "tutorial"]
title: 'Training Your Own LLM Model'
---

# Training your own LLM Model

**A practical, open-source-only roadmap that runs on a MacBook and deploys to a cheap Linux laptop.**

---

## Why This Path Works

You don't need closed-source APIs or a $10k GPU to train a model. Take an open model, adapt it to a real problem, measure it honestly, and deploy it cheaply. That's exactly what this guide does.

By the end you'll have a **fine-tuned Text-to-SQL model**, quantized to 4-bit, served from commodity hardware, with retrieval-augmented generation on top and hard evaluation numbers to prove it works. That's a portfolio piece that speaks louder than any certificate.

---

## Some considerations I am making

- You already know basics of LLM
- You already know basic Python scripting
- Some parts of this content is created using AI; my code is messy!
- Whatever you see here is from public sources. The content and views here are my own and not f my employer.

---

## Which Laptop Am I Using

** I am using the M2 Pro (16 GB) for training. It's not close.**

The M2's unified memory combined with PyTorch's MPS (Metal) backend lets you fine-tune 1B–3B parameter models with LoRA in reasonable time. An Ubuntu laptop with weak graphics would be CPU-only — 20–50× slower and not realistic for fine-tuning.

I also have an Ubuntu laptop for **deployment**. We'll use it in Part 2 to prove you can ship models on cheap commodity hardware.

---

## The Project: Text-to-SQL Fine-Tuning

**What you'll build:** Fine-tune a small open-source LLM (`Qwen2.5-1.5B-Instruct`) on the Spider dataset so it converts natural-language questions into SQL queries. Then deploy it as an API + chat UI, quantize it for cheap hardware, and add retrieval on top.

### Some Real Life Learning Impact From This

- **Text-to-SQL is a real production problem.** Every data team wants "ask our database in plain English."
- **It has an objective evaluation metric**, does the generated SQL execute and return the correct result? So you can report hard numbers.
- **It demonstrates the full ML pipeline:** data → LoRA fine-tuning → evaluation → deployment → quantization → RAG.
- **It shows you can make small models punch above their weight**

> **Alternatives you can swap in later:** structured JSON extraction from invoices, GitHub issue auto-triager, domain-specific code assistant (Terraform, dbt, SQL dialects).

---

## Part 1: Fine-Tune on the M2 Pro

### Step 1: Environment Setup

Install Homebrew if you don't have it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Install Python, git, and uv (fast modern package manager):

```bash
brew install python@3.11 uv git
```

Create your project:

```bash
mkdir text2sql-llm && cd text2sql-llm
uv venv --python 3.11
source .venv/bin/activate
```

Install the stack:

```bash
uv pip install torch torchvision torchaudio
uv pip install transformers datasets accelerate peft trl
uv pip install sqlparse gradio fastapi uvicorn
uv pip install sentencepiece protobuf
```

Verify MPS (Apple GPU) works:

```bash
python -c "import torch; print('MPS available:', torch.backends.mps.is_available())"
```

Expect: `MPS available: True`.

### Step 2: Model and Dataset

- **Model:** `Qwen/Qwen2.5-1.5B-Instruct` — strong reasoning, Apache 2.0, fits in 16 GB with LoRA.
- **Dataset:** `spider` on Hugging Face — ~10k questions paired with SQL across 200 databases.

```bash
huggingface-cli download Qwen/Qwen2.5-1.5B-Instruct --local-dir ./model
python -c "from datasets import load_dataset; load_dataset('spider')"
```

### Step 3: Training Script

Create `train.py`:

```python
import torch
from datasets import load_dataset
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig, get_peft_model
from trl import SFTTrainer, SFTConfig

MODEL = "./model"
tokenizer = AutoTokenizer.from_pretrained(MODEL)
model = AutoModelForCausalLM.from_pretrained(
    MODEL, torch_dtype=torch.float16, device_map="mps"
)

# LoRA: train ~1% of parameters
lora = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    bias="none", task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora)

ds = load_dataset("spider", split="train")

def format_example(row):
    return {"text": (
        f"You are a SQL expert. Given a database schema and a question, write the SQL query.\n"
        f"Schema: {row['db_id']}\n"
        f"Question: {row['question']}\n"
        f"SQL: {row['query']}"
    )}

ds = ds.map(format_example)

args = SFTConfig(
    output_dir="./checkpoints",
    per_device_train_batch_size=2,
    gradient_accumulation_steps=4,
    num_train_epochs=2,
    learning_rate=2e-4,
    logging_steps=20,
    save_steps=200,
    optim="adamw_torch",
    max_seq_length=1024,
)

trainer = SFTTrainer(model=model, args=args, train_dataset=ds, tokenizer=tokenizer)
trainer.train()
model.save_pretrained("./text2sql-qwen-lora")
```

Run it:

```bash
python train.py
```

Expect 3–6 hours for 2 epochs on M2 Pro. Run it overnight.

### Step 4: Evaluation (The Part Most Beginners Skip)

Create `evaluate.py`, generate SQL, execute it against real SQLite databases, compare results to ground truth:

```python
import sqlite3
from datasets import load_dataset
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import PeftModel
import torch

base = AutoModelForCausalLM.from_pretrained("./model", torch_dtype=torch.float16, device_map="mps")
model = PeftModel.from_pretrained(base, "./text2sql-qwen-lora")
tok = AutoTokenizer.from_pretrained("./model")

dev = load_dataset("spider", split="validation")

correct = 0
for row in dev.select(range(200)):
    prompt = f"Schema: {row['db_id']}\nQuestion: {row['question']}\nSQL:"
    ids = tok(prompt, return_tensors="pt").to("mps")
    out = model.generate(**ids, max_new_tokens=128, do_sample=False)
    pred_sql = tok.decode(out[0], skip_special_tokens=True).split("SQL:")[-1].strip()

    db = f"spider/database/{row['db_id']}/{row['db_id']}.sqlite"
    try:
        con = sqlite3.connect(db)
        pred_rows = con.execute(pred_sql).fetchall()
        gold_rows = con.execute(row["query"]).fetchall()
        if set(pred_rows) == set(gold_rows):
            correct += 1
    except Exception:
        pass

print(f"Execution accuracy: {correct/200:.2%}")
```

**Target:** 40–55% execution accuracy. Beating the base (un-fine-tuned) model by 15+ points is a great achievement.

### Step 5: Demo UI

Create `app.py`:

```python
import gradio as gr
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import PeftModel
import torch

base = AutoModelForCausalLM.from_pretrained("./model", torch_dtype=torch.float16, device_map="mps")
model = PeftModel.from_pretrained(base, "./text2sql-qwen-lora")
tok = AutoTokenizer.from_pretrained("./model")

def generate_sql(schema, question):
    prompt = f"Schema: {schema}\nQuestion: {question}\nSQL:"
    ids = tok(prompt, return_tensors="pt").to("mps")
    out = model.generate(**ids, max_new_tokens=128, do_sample=False)
    return tok.decode(out[0], skip_special_tokens=True).split("SQL:")[-1].strip()

gr.Interface(
    fn=generate_sql,
    inputs=["text", "text"],
    outputs="text",
    title="Text-to-SQL (Fine-tuned Qwen2.5-1.5B)",
).launch(share=True)
```

---

## Part 2: Quantize to 4-bit GGUF and Deploy on Your Ubuntu Laptop

**This mirrors how real teams might ship LLMs to edge devices, self-hosted servers, or low-cost cloud VMs.

### Step 1: Merge the LoRA Adapter into the Base Model (on Mac)

`llama.cpp` converts full models, not LoRA adapters, so merge first. Create `merge.py`:

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel

base = AutoModelForCausalLM.from_pretrained(
    "./model", torch_dtype=torch.float16
)
model = PeftModel.from_pretrained(base, "./text2sql-qwen-lora")
model = model.merge_and_unload()

model.save_pretrained("./text2sql-merged")
tok = AutoTokenizer.from_pretrained("./model")
tok.save_pretrained("./text2sql-merged")

print("Merged model saved to ./text2sql-merged")
```

```bash
python merge.py
```

### Step 2: Clone and Build llama.cpp (on Mac)

```bash
cd ..
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
```

Build it on Mac, Metal support is enabled by default:

```bash
cmake -B build
cmake --build build --config Release -j
```

Install the Python conversion dependencies:

```bash
uv pip install -r requirements.txt
```

### Step 3: Convert HF Model → GGUF (FP16)

```bash
python convert_hf_to_gguf.py ../text2sql-llm/text2sql-merged \
    --outfile ../text2sql-llm/text2sql-f16.gguf \
    --outtype f16
```

You now have a ~3 GB FP16 GGUF file.

### Step 4: Quantize to 4-bit

The `llama-quantize` binary lives in `build/bin/`:

```bash
./build/bin/llama-quantize \
    ../text2sql-llm/text2sql-f16.gguf \
    ../text2sql-llm/text2sql-q4_k_m.gguf \
    Q4_K_M
```

**Quantization format guide:**

| Format | Notes |
|---|---|
| `Q4_K_M` | Best balance of size/quality for most cases. ~0.9 GB for a 1.5B model. |
| `Q5_K_M` | Slightly larger, slightly better quality. |
| `Q8_0` | Near-lossless, ~2× the size of Q4. |

For a 1.5B model, `Q4_K_M` typically loses <2% accuracy versus FP16. Quick sanity-check on the Mac:

```bash
./build/bin/llama-cli -m ../text2sql-llm/text2sql-q4_k_m.gguf \
    -p "Schema: concert_singer\nQuestion: How many singers are there?\nSQL:" \
    -n 64
```

### Step 5: Transfer the GGUF to Your Ubuntu Laptop

```bash
scp ../text2sql-llm/text2sql-q4_k_m.gguf your-user@ubuntu-laptop-ip:~/
```

Or use `rsync`, a USB drive, or upload it to Hugging Face Hub as a private repo.

### Step 6: Install llama.cpp on Ubuntu

```bash
sudo apt update
sudo apt install -y build-essential cmake git curl
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
cmake -B build
cmake --build build --config Release -j$(nproc)
```

No GPU flags needed, CPU build is fine. The 4-bit 1.5B model will run at ~10–30 tokens/second on a modest CPU, which is very usable.

### Step 7: Run Inference on Ubuntu

Interactive CLI:

```bash
./build/bin/llama-cli -m ~/text2sql-q4_k_m.gguf \
    -p "Schema: concert_singer\nQuestion: How many singers are there?\nSQL:" \
    -n 128 --temp 0
```

Or run as an OpenAI-compatible HTTP server:

```bash
./build/bin/llama-server -m ~/text2sql-q4_k_m.gguf --port 8080
```

Hit it with curl:

```bash
curl http://localhost:8080/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "messages": [
        {"role": "user", "content": "Schema: concert_singer\nQuestion: How many singers are there?\nSQL:"}
      ],
      "temperature": 0,
      "max_tokens": 128
    }'
```

Re-run your evaluation script against this endpoint and compare the quantized model's execution accuracy to the FP16 version. Report both numbers.

---

## Part 3: Add Retrieval-Augmented Generation (RAG)

**The problem this solves:** Your model was trained on 200 Spider databases. When a user asks about a database it's never seen, it has no idea what tables or columns exist. RAG fixes this by retrieving the relevant schema at query time and injecting it into the prompt.

### Architecture

```
User question
      │
      ▼
┌─────────────────────┐
│  Embedding model    │  (sentence-transformers)
└─────────┬───────────┘
          │ query vector
          ▼
┌─────────────────────┐
│    FAISS index      │  (schema vectors + metadata)
└─────────┬───────────┘
          │ top-k schemas
          ▼
┌─────────────────────┐
│  Prompt builder     │  (schema + question)
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ Your fine-tuned LLM │  (generates SQL)
└─────────────────────┘
```

### Step 1: Install the RAG Stack

```bash
uv pip install sentence-transformers faiss-cpu
```

On the Ubuntu deployment machine too — same command.

### Step 2: Build a Schema Corpus

For each database, write a human-readable schema description. Create `build_corpus.py`:

```python
import json
import sqlite3
from pathlib import Path

DB_ROOT = Path("spider/database")
corpus = []

for db_dir in DB_ROOT.iterdir():
    sqlite_file = db_dir / f"{db_dir.name}.sqlite"
    if not sqlite_file.exists():
        continue

    con = sqlite3.connect(sqlite_file)
    cur = con.cursor()
    tables = cur.execute(
        "SELECT name FROM sqlite_master WHERE type='table'"
    ).fetchall()

    schema_parts = []
    for (tname,) in tables:
        cols = cur.execute(f"PRAGMA table_info({tname})").fetchall()
        col_desc = ", ".join(f"{c[1]} {c[2]}" for c in cols)
        schema_parts.append(f"Table {tname}({col_desc})")

    schema_text = "; ".join(schema_parts)
    corpus.append({
        "db_id": db_dir.name,
        "schema": schema_text,
        "doc": f"Database {db_dir.name}: {schema_text}",
    })
    con.close()

with open("schema_corpus.json", "w") as f:
    json.dump(corpus, f, indent=2)

print(f"Wrote {len(corpus)} schemas")
```

```bash
python build_corpus.py
```

### Step 3: Embed and Build the FAISS Index

Create `build_index.py`:

```python
import json
import numpy as np
import faiss
from sentence_transformers import SentenceTransformer

with open("schema_corpus.json") as f:
    corpus = json.load(f)

# Small, fast, good-quality embedder. 384-dim vectors.
embedder = SentenceTransformer("all-MiniLM-L6-v2")

docs = [item["doc"] for item in corpus]
embeddings = embedder.encode(
    docs, batch_size=32, show_progress_bar=True, normalize_embeddings=True
).astype("float32")

# Inner product on normalized vectors == cosine similarity
index = faiss.IndexFlatIP(embeddings.shape[1])
index.add(embeddings)

faiss.write_index(index, "schema.faiss")
with open("schema_meta.json", "w") as f:
    json.dump(corpus, f)

print(f"Indexed {len(corpus)} schemas, dim={embeddings.shape[1]}")
```

**Why `all-MiniLM-L6-v2`:** it's 80 MB, runs on CPU in milliseconds, and is the industry-default starter embedder. When you're ready to upgrade, swap in `BAAI/bge-small-en-v1.5` — one-line change, usually a few points of recall better.

### Step 4: Build the RAG Pipeline

Create `rag_app.py`:

```python
import json
import faiss
import gradio as gr
from sentence_transformers import SentenceTransformer
from transformers import AutoTokenizer, AutoModelForCausalLM
from peft import PeftModel
import torch

# --- Retriever ---
embedder = SentenceTransformer("all-MiniLM-L6-v2")
index = faiss.read_index("schema.faiss")
with open("schema_meta.json") as f:
    corpus = json.load(f)

def retrieve(question, k=3):
    q = embedder.encode([question], normalize_embeddings=True).astype("float32")
    scores, idxs = index.search(q, k)
    return [(corpus[i], float(s)) for i, s in zip(idxs[0], scores[0])]

# --- Generator ---
base = AutoModelForCausalLM.from_pretrained(
    "./model", torch_dtype=torch.float16, device_map="mps"
)
model = PeftModel.from_pretrained(base, "./text2sql-qwen-lora")
tok = AutoTokenizer.from_pretrained("./model")

def generate_sql(question):
    hits = retrieve(question, k=3)
    top_schema = hits[0][0]["schema"]
    prompt = (
        f"You are a SQL expert. Use the schema to answer.\n"
        f"Schema: {top_schema}\n"
        f"Question: {question}\n"
        f"SQL:"
    )
    ids = tok(prompt, return_tensors="pt").to("mps")
    out = model.generate(**ids, max_new_tokens=128, do_sample=False)
    sql = tok.decode(out[0], skip_special_tokens=True).split("SQL:")[-1].strip()
    retrieved_info = "\n".join(
        f"  {i+1}. {h[0]['db_id']} (score={h[1]:.3f})"
        for i, h in enumerate(hits)
    )
    return sql, retrieved_info

with gr.Blocks(title="Text-to-SQL + RAG") as demo:
    gr.Markdown("# Text-to-SQL with RAG")
    q = gr.Textbox(label="Ask a question in plain English")
    btn = gr.Button("Generate SQL")
    sql_out = gr.Textbox(label="Generated SQL")
    retr_out = gr.Textbox(label="Retrieved databases (top-3)")
    btn.click(generate_sql, inputs=q, outputs=[sql_out, retr_out])

demo.launch(share=True)
```

```bash
python rag_app.py
```

### Step 5: Evaluate the RAG System (Very Important)

Create `eval_rag.py` that measures two things:

1. **Retrieval recall@k** — is the correct `db_id` in the top-k?
2. **End-to-end execution accuracy** — does the generated SQL return the right rows?

```python
import json, sqlite3
from datasets import load_dataset
from rag_app import retrieve, generate_sql  # reuse pipeline

dev = load_dataset("spider", split="validation").select(range(200))

recall_at_3, exec_correct = 0, 0

for row in dev:
    hits = retrieve(row["question"], k=3)
    retrieved_ids = [h[0]["db_id"] for h in hits]
    if row["db_id"] in retrieved_ids:
        recall_at_3 += 1

    sql, _ = generate_sql(row["question"])
    db = f"spider/database/{row['db_id']}/{row['db_id']}.sqlite"
    try:
        con = sqlite3.connect(db)
        pred = con.execute(sql).fetchall()
        gold = con.execute(row["query"]).fetchall()
        if set(pred) == set(gold):
            exec_correct += 1
    except Exception:
        pass

n = len(dev)
print(f"Retrieval Recall@3: {recall_at_3/n:.2%}")
print(f"End-to-end execution accuracy: {exec_correct/n:.2%}")
```

Report both numbers in your README. This breakdown "retrieval is the bottleneck" vs "generation is the bottleneck", is the exact kind of analysis senior engineers do.

### Step 6: Production-Minded RAG Upgrades For Future

Pick one to implement, each shows a different facet of RAG maturity:

- **Hybrid search:** combine FAISS (semantic) with BM25 (keyword) using `rank-bm25`. Reranking frequently adds 5–10 points of recall.
- **Cross-encoder reranker:** take top-20 from FAISS, rerank with `cross-encoder/ms-marco-MiniLM-L-6-v2`, keep top-3. Usually a big quality win.
- **Chunking strategy:** if schemas are huge, split them into table-level chunks with metadata, then retrieve tables instead of whole databases.
- **Query rewriting:** use the LLM itself to rephrase ambiguous questions before retrieval.

---


## Where to Go Next

Some other ideas we can implement:

- **Structured extraction:** Fine-tune on invoice/receipt data → JSON output. Add a Gradio drag-and-drop UI.
- **Code assistant for a niche:** Fine-tune on Terraform, dbt, or GraphQL. Ship as a VSCode extension.
- **Evaluation framework:** Build a small harness that scores LLM outputs on domain tasks. Show you understand measurement, not just training.

---

## References & Resources

### Models

- **Qwen2.5-1.5B-Instruct**: The base model used in this guide. Apache 2.0 licensed, 1.54B parameters, 32k context window.
  - Model card: [huggingface.co/Qwen/Qwen2.5-1.5B-Instruct](https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct)
  - Qwen2.5 technical report: [arxiv.org/abs/2407.10671](https://arxiv.org/abs/2407.10671)

- **all-MiniLM-L6-v2**: The embedding model used for RAG retrieval. 384-dimensional vectors, 80 MB, runs on CPU.
  - Model card: [huggingface.co/sentence-transformers/all-MiniLM-L6-v2](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2)

### Datasets

- **Spider**: A large-scale, cross-domain text-to-SQL benchmark with 10,181 questions across 200 databases. Created by 11 Yale students.
  - Project page: [yale-lily.github.io/spider](https://yale-lily.github.io/spider)
  - Paper (EMNLP 2018): Yu et al., *"Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and Text-to-SQL Task"* — [arxiv.org/abs/1809.08887](https://arxiv.org/abs/1809.08887)
  - Hugging Face dataset: [huggingface.co/datasets/xlangai/spider](https://huggingface.co/datasets/xlangai/spider)
  - GitHub: [github.com/taoyds/spider](https://github.com/taoyds/spider)

### Key Papers & Techniques

- **LoRA (Low-Rank Adaptation)**: The parameter-efficient fine-tuning method used throughout this guide. Freezes pre-trained weights and injects trainable low-rank decomposition matrices.
  - Paper (ICLR 2022): Hu et al., *"LoRA: Low-Rank Adaptation of Large Language Models"*: [arxiv.org/abs/2106.09685](https://arxiv.org/abs/2106.09685)
  - Original implementation: [github.com/microsoft/LoRA](https://github.com/microsoft/LoRA)

### Libraries & Tools

- **Hugging Face Transformers**: The core library for loading and working with pre-trained models.
  - Docs: [huggingface.co/docs/transformers](https://huggingface.co/docs/transformers)
  - GitHub: [github.com/huggingface/transformers](https://github.com/huggingface/transformers)

- **PEFT (Parameter-Efficient Fine-Tuning)**: Hugging Face's library for LoRA, prefix tuning, and other adapter methods.
  - Docs: [huggingface.co/docs/peft](https://huggingface.co/docs/peft)
  - GitHub: [github.com/huggingface/peft](https://github.com/huggingface/peft)

- **TRL (Transformer Reinforcement Learning)**: Provides `SFTTrainer` used for supervised fine-tuning in this guide.
  - Docs: [huggingface.co/docs/trl](https://huggingface.co/docs/trl)
  - GitHub: [github.com/huggingface/trl](https://github.com/huggingface/trl)

- **llama.cpp**: C/C++ inference engine for running LLMs locally. Handles GGUF conversion and quantization.
  - GitHub: [github.com/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)

- **FAISS (Facebook AI Similarity Search)**: Library for efficient similarity search and clustering of dense vectors. Used for the RAG index.
  - GitHub: [github.com/facebookresearch/faiss](https://github.com/facebookresearch/faiss)
  - Wiki & tutorials: [github.com/facebookresearch/faiss/wiki](https://github.com/facebookresearch/faiss/wiki)
  - Paper: Johnson et al., *"Billion-scale similarity search with GPUs"* (IEEE Transactions on Big Data, 2019)

- **Sentence Transformers**: Library for computing sentence embeddings, used to build the FAISS index.
  - GitHub: [github.com/UKPLab/sentence-transformers](https://github.com/UKPLab/sentence-transformers)
  - Docs: [sbert.net](https://www.sbert.net/)

- **Gradio**: Python library for building ML demo UIs. Powers the chat interface in this guide.
  - Docs: [gradio.app](https://www.gradio.app/)
  - GitHub: [github.com/gradio-app/gradio](https://github.com/gradio-app/gradio)

- **PyTorch MPS Backend**: Apple's Metal Performance Shaders backend for PyTorch, enabling GPU acceleration on Apple Silicon.
  - Docs: [pytorch.org/docs/stable/notes/mps.html](https://pytorch.org/docs/stable/notes/mps.html)

- **uv**: Fast Python package manager used for environment setup.
  - GitHub: [github.com/astral-sh/uv](https://github.com/astral-sh/uv)

### Further Reading

- Hugging Face LLM course (free): covers fine-tuning, RAG, and deployment: [huggingface.co/learn/llm-course](https://huggingface.co/learn/llm-course)
- Quantization guide with GGUF and llama.cpp by Maxime Labonne: [towardsdatascience.com/quantize-llama-models-with-ggml-and-llama-cpp](https://towardsdatascience.com/quantize-llama-models-with-ggml-and-llama-cpp-3612dfbcc172/)
- PEFT blog post by Hugging Face: introduction and practical examples: [huggingface.co/blog/peft](https://huggingface.co/blog/peft)