# Using AI and LLM Inference on NRP

NRP runs hundreds of NVIDIA GPUs (and Qualcomm Cloud AI 100 cards) across the country. A subset of those GPUs power a community-shared, OpenAI-compatible LLM inference endpoint at **`https://ellm.nrp-nautilus.io/v1`**. This tutorial walks through the full path: connect to that endpoint from Jupyter AI, `curl`, Python, and an agentic coding CLI (opencode), then bring up your own GPU pod for ad-hoc inference and a RAG pipeline against the [Milvus](https://milvus.io) vector database NRP also runs.

**Conventions.** Examples use the **`nrp-training-k8s`** namespace and the workshop A10 reservation (label `nrp-training=true`, taint `nautilus.io/reservation=nrp:NoSchedule`). Replace **`<username>`** with your NRP/GitHub username in any YAML or pod name. Manifests live in [`yamls/`](yamls).

> 📘 **Docs:** [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/) · [Managed LLMs](https://nrp.ai/documentation/userdocs/ai/llm-managed/) · [Available models](https://nrp.ai/documentation/userdocs/ai/llm-managed/models/) · [LLM API access](https://nrp.ai/documentation/userdocs/ai/llm-managed/api-access/) · [Client configs](https://nrp.ai/documentation/userdocs/ai/llm-managed/client-configs/) · [Vector DB (Milvus)](https://nrp.ai/documentation/userdocs/vector-database/) · [LLM token](https://nrp.ai/llmtoken)

---

## 1. NRP GPUs power a managed LLM service

NRP exposes GPUs to users in two complementary ways:

1. **Bring-your-own pod** — request `nvidia.com/gpu` (or model-specific keys like `nvidia.com/a100`, `nvidia.com/h200`, `nvidia.com/gh200`) in your container's `resources.limits`, optionally pin to a product with `nvidia.com/gpu.product` node-affinity. See §6 for hands-on. Full reference: [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/).
2. **Managed LLM service** — a rotating catalog of open-weights LLMs is hosted on those same GPUs and exposed behind the OpenAI-compatible URL `https://ellm.nrp-nautilus.io/v1`. You don't run a pod or pay GPU time; you just send HTTP requests with a bearer token from [https://nrp.ai/llmtoken](https://nrp.ai/llmtoken). See [Managed LLMs](https://nrp.ai/documentation/userdocs/ai/llm-managed/).

**Available models** (see [models page](https://nrp.ai/documentation/userdocs/ai/llm-managed/models/) for the live list — capabilities below are at workshop time):

| Model | Status | Params | Context | Tools | Reasoning | Vision |
|---|---|---|---|---|---|---|
| `qwen3` | main | 397B (A17B active) | 262K | ✓ | ✓ | image, video |
| `qwen3-small` | main | 27B | 262K | ✓ | ✓ | image, video |
| `gpt-oss` | main | 120B | 131K | ✓ | ✓ | — |
| `gemma` | main | 31B | 262K | ✓ | ✓ | image, video |
| `minimax-m2` | main | 230B | 204K | ✓ | ✓ | — |
| `qwen3-embedding` | main | 8B | — | embeddings only | — | — |
| `glm-4.7` | evaluating | 358B | 202K | ✓ | ✓ | — |
| `kimi` | evaluating | 1T MoE | 262K | ✓ | ✓ | image, video |
| `olmo` | evaluating | 32B | 64K | ✓ | — | — |

**Browser entry points** (no token needed for the UI; sign in with NRP):

- [https://nrp-openwebui.nrp-nautilus.io](https://nrp-openwebui.nrp-nautilus.io) — Open WebUI
- [https://librechat.nrp-nautilus.io](https://librechat.nrp-nautilus.io) — LibreChat

> 🔑 **Workshop token & endpoint.** Every example below uses these two values verbatim — copy them once:
>
> - **API base URL:** `https://ellm.nrp-nautilus.io/v1`
> - **Bearer token:** `rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4`
>
> They are already exported as `OPENAI_API_BASE` / `OPENAI_API_KEY` inside the workshop JupyterHub and inside any pod that mounts the `nrp-llm-token` Secret in `nrp-training-k8s`. After the workshop, mint your own token at [https://nrp.ai/llmtoken](https://nrp.ai/llmtoken) and substitute it in.

---

## 2. Talk to the LLM from Jupyter AI

The Training JupyterHub at [training.nrp-nautilus.io](https://training.nrp-nautilus.io) ships with [Jupyter AI](https://jupyter-ai.readthedocs.io/) **pre-configured for the NRP managed LLM** — provider `openai-chat:minimax-m2`, embeddings provider `openai:qwen3-embedding`, and the workshop token wired in. There is nothing to install.

**Try it now — chat panel (recommended).** Click the **chat (robot) icon** in the left sidebar. The settings are already populated; type a question into the prompt at the bottom and hit send. The chat replies stream back from `minimax-m2` running on NRP GPUs.

**Or use the cell magic.** Spawn a session, open a Python 3 notebook, then:

```python
%load_ext jupyter_ai_magics
```

```
%%ai openai-chat:minimax-m2
What is the National Research Platform in two sentences?
```

Switch model per cell — first line of the magic is `%%ai <provider>:<model>`:

```
%%ai openai-chat:gpt-oss
Write a Python one-liner that prints the first 20 Fibonacci numbers.
```

`%ai list` shows every registered provider. Both the chat panel and the magic share the same `~/.local/share/jupyter/jupyter_ai/config.json`, so picking a different model in one is reflected in the other.

**What you learn.** Jupyter AI is the lowest-friction way to demo the managed LLM during a class — students log in, no token handoff, no `pip install`. Configuration lives in `~/.local/share/jupyter/jupyter_ai/config.json`; the JupyterHub spawner writes it for you on every spawn.

---

## 3. Talk to the LLM with `curl`

The endpoint is OpenAI-compatible — anything that speaks OpenAI's REST API speaks NRP. Open a terminal in JupyterLab (or any shell with `curl`).

**List models:**

```bash
curl -s -H "Authorization: Bearer rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4" \
     https://ellm.nrp-nautilus.io/v1/models | python3 -m json.tool | head -30
```

**Expected output:**

```
{
    "data": [
        {"id": "gpt-oss",      "created": ..., "object": "model", "owned_by": "SDSC"},
        {"id": "gemma",        "created": ..., "object": "model", "owned_by": "SDSC"},
        {"id": "minimax-m2",   "created": ..., "object": "model", "owned_by": "NRP"},
        {"id": "qwen3",        "created": ..., "object": "model", "owned_by": "NRP"},
        ...
    ],
    "object": "list"
}
```

**Send a chat completion:**

```bash
curl -s -X POST https://ellm.nrp-nautilus.io/v1/chat/completions \
  -H "Authorization: Bearer rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "minimax-m2",
    "messages": [
      {"role": "system", "content": "Answer in one sentence."},
      {"role": "user",   "content": "What is the National Research Platform?"}
    ]
  }' | python3 -c 'import json,sys; print(json.load(sys.stdin)["choices"][0]["message"]["content"])'
```

**Stream tokens** (add `"stream": true` and read SSE chunks):

```bash
curl -sN -X POST https://ellm.nrp-nautilus.io/v1/chat/completions \
  -H "Authorization: Bearer rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4" \
  -H "Content-Type: application/json" \
  -d '{"model":"minimax-m2","stream":true,"messages":[{"role":"user","content":"Count 1 to 5 with a brief reason for each."}]}'
```

You'll see `data: {…}` lines arrive incrementally, ending with `data: [DONE]`.

**What you learn.** `curl` is the universal smoke test — if it works here, *anything* OpenAI-compatible (Cherry Studio, LangChain, openai-python, your own app) works.

---

## 4. Talk to the LLM with Python (`openai` SDK)

Install the official OpenAI Python client (already present on JupyterHub spawns; outside, `pip install openai`):

```bash
python3 -c 'import openai; print(openai.__version__)'
```

**Single completion:**

```python
from openai import OpenAI

client = OpenAI(
    api_key="rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4",
    base_url="https://ellm.nrp-nautilus.io/v1",
)

resp = client.chat.completions.create(
    model="minimax-m2",
    messages=[
        {"role": "system", "content": "You are a concise teaching assistant."},
        {"role": "user",   "content": "Explain Kubernetes namespaces in two sentences."},
    ],
)
print(resp.choices[0].message.content)
```

**Streaming:**

```python
from openai import OpenAI

client = OpenAI(
    api_key="rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4",
    base_url="https://ellm.nrp-nautilus.io/v1",
)

stream = client.chat.completions.create(
    model="minimax-m2",
    messages=[{"role": "user", "content": "Write a haiku about GPUs."}],
    stream=True,
)
for chunk in stream:
    if not chunk.choices:
        continue
    delta = chunk.choices[0].delta.content
    if delta:
        print(delta, end="", flush=True)
print()
```

**Pick a different model** by changing `model="…"`. Use `gpt-oss` for code-heavy tasks, `qwen3` for the largest context window (262K), `gemma` for vision-language input.

**What you learn.** The exact same code targets the OpenAI cloud, NRP's managed LLM, or any vLLM/TGI server you bring up yourself (§6) — only `base_url` changes. This is the portability dividend of the OpenAI-compatible API.

---

## 5. Agentic coding with `opencode` — build a chess game

[`opencode`](https://opencode.ai) is a TUI agentic coding tool, similar to Claude Code or Cursor's CLI. It plans, edits files, runs tools, and iterates. We'll point it at the NRP managed LLM and have it write a small chess game from scratch.

**Install** (inside a JupyterLab terminal — works in macOS/Linux too):

```bash
curl -fsSL https://opencode.ai/install | bash
export PATH="$HOME/.opencode/bin:$PATH"
opencode --version
```

**Configure** the NRP provider. Write `~/.config/opencode/opencode.json`:

```bash
mkdir -p ~/.config/opencode
cat > ~/.config/opencode/opencode.json <<'JSON'
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "nrp": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "NRP LLM",
      "options": {
        "baseURL": "https://ellm.nrp-nautilus.io/v1",
        "apiKey": "rifgnLi8QEfRECOgFKVFHaeTLBeSogQ4"
      },
      "models": {
        "minimax-m2": { "name": "MiniMax M2"  },
        "gpt-oss":    { "name": "GPT-OSS"     },
        "qwen3":      { "name": "Qwen3 397B"  },
        "gemma":      { "name": "Gemma 31B"   },
        "kimi":       { "name": "Kimi 1T MoE" }
      }
    }
  },
  "model": "nrp/minimax-m2"
}
JSON
```

The `apiKey` value above is the workshop bearer token from §1 — paste it verbatim. After the workshop, replace it with your own from [https://nrp.ai/llmtoken](https://nrp.ai/llmtoken) (or rewrite as `"{env:OPENAI_API_KEY}"` to read from the environment).

**Make a project directory and launch opencode:**

```bash
mkdir -p ~/chess && cd ~/chess
opencode
```

This drops you into the opencode TUI. Press `/` to open the prompt and paste:

```
Write a single-file Python program chess_game.py that lets two humans play
chess in the terminal. Use the python-chess library. Render the board after
every move using board.unicode(). Accept moves in SAN (e.g., "e4", "Nf3").
When the game ends, print the result. Add a top-of-file docstring. After
writing the file, add a requirements.txt pinning python-chess to 1.999, and
tell me the exact commands to install and play.
```

opencode plans, writes `chess_game.py` and `requirements.txt`, and prints the run instructions. **Run them:**

```bash
pip install -r requirements.txt
python chess_game.py
```

> ⚠️ **Don't name the file `chess.py`** — it shadows the `python-chess` package. `import chess` then re-imports your script and `chess.Board()` raises `AttributeError`. Models will sometimes pick `chess.py` anyway because the prompt says "chess game"; if that happens, just rename. Also, models have been known to invent versions like `python-chess==1.10.0` that do not exist on PyPI — the actual current pin is `python-chess==1.999`. The instruction above pre-pins to avoid the round-trip; if you skipped it, fix `requirements.txt` by hand and re-`pip install`.

Try a few moves (`e4`, `e5`, `Nf3`, …). Press `Ctrl+C` to quit.

**Switch models inside opencode** with `Ctrl+P` → Switch models — try the same prompt against `gpt-oss` (better at code than minimax often) or `qwen3` (longest context).

**What you learn.** Any agentic coding tool that supports an OpenAI-compatible base URL — opencode, Crush, Continue, Cursor's custom-provider field, Claude Code via `ANTHROPIC_BASE_URL` — works against NRP. You bring the workflow you already use; NRP supplies the inference. Reference: [client configs](https://nrp.ai/documentation/userdocs/ai/llm-managed/client-configs/).

---

## 6. Bring your own GPU pod

The managed LLM is convenient but constrained: you don't pick the model weights, the version, the quantization, or the runtime. When you need that control — or when you want to run training, embeddings, or a custom inference engine — request a GPU yourself. All examples below already include the workshop reservation pattern (you saw this in Tutorial 1):

```yaml
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - { key: nrp-training, operator: In, values: ["true"] }
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - { key: nvidia.com/gpu.product, operator: In, values: [NVIDIA-A10] }
tolerations:
- { key: nautilus.io/reservation, operator: Equal, value: nrp, effect: NoSchedule }
```

The **toleration** lets your pod land on the tainted reservation nodes; the **required affinity** constrains GPUs to A10s; the **preferred affinity** prefers our reserved pool. Outside this workshop, drop both blocks (or change them) to land on general NRP capacity. See [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/) for other GPU products and the matching resource keys.

### 6.1 PyTorch GPU sanity check + 1-epoch MNIST

`yamls/pytorch-training.yaml` requests **1 NVIDIA A10**, runs `nvidia-smi`, then trains MNIST for one epoch using `nvcr.io/nvidia/pytorch:23.07-py3`. Apply (replace `<username>`):

```bash
kubectl apply -n nrp-training-k8s -f yamls/pytorch-training.yaml
kubectl get pod -n nrp-training-k8s tutorial-<username>-gp3 -w
```

Once `Completed`, read the logs:

```bash
kubectl logs -n nrp-training-k8s tutorial-<username>-gp3 | tail -25
```

**Expected output (truncated):**

```
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.86.10              Driver Version: 535.86.10    CUDA Version: 12.2     |
+-----------------------------------------+----------------------+----------------------+
|   0  NVIDIA A10                  ...    |  ...                 |   0%      Default    |
+-----------------------------------------+----------------------+----------------------+

Train Epoch: 1 [0/60000 (0%)]    Loss: 2.305199
...
Test set: Average loss: 0.0501, Accuracy: 9849/10000 (98%)
PyTorch MNIST completed successfully.
```

Cleanup:

```bash
kubectl delete -n nrp-training-k8s -f yamls/pytorch-training.yaml
```

### 6.2 Run your own LLM with TGI (Text Generation Inference)

Same pattern but with HuggingFace's [TGI](https://github.com/huggingface/text-generation-inference) serving `HuggingFaceH4/zephyr-7b-beta` on a single A10. `yamls/tgi-inference.yaml` boots the server and keeps it alive for one hour.

```bash
kubectl apply -n nrp-training-k8s -f yamls/tgi-inference.yaml
kubectl get pod -n nrp-training-k8s tutorial-<username>-tgi -w
```

Wait for the model to download and the server to log `Connected` (1–3 min). Then **port-forward** the TGI port to your machine:

```bash
kubectl port-forward -n nrp-training-k8s tutorial-<username>-tgi 8080:80
```

In another terminal:

```bash
curl -s http://127.0.0.1:8080/info | python3 -m json.tool | head
curl -s http://127.0.0.1:8080/generate \
  -H "Content-Type: application/json" \
  -d '{"inputs":"Why are penguins black and white?","parameters":{"max_new_tokens":60}}'
```

Or — and this is the punchline — point the **same OpenAI-style code from §4** at it (TGI exposes an OpenAI-compatible `/v1/chat/completions`). In Python:

```python
from openai import OpenAI
client = OpenAI(api_key="not-needed", base_url="http://127.0.0.1:8080/v1")
print(client.chat.completions.create(
    model="tgi",
    messages=[{"role":"user","content":"Hi from my own GPU."}],
).choices[0].message.content)
```

Cleanup:

```bash
kubectl delete -n nrp-training-k8s -f yamls/tgi-inference.yaml
```

### 6.3 RAG over the NRP docs — managed LLM **vs** a local Ollama

This exercise has the highest density of any in this tutorial: a managed vector database, an OpenAI-compatible managed LLM, and an offline LLM running on your own GPU — all wired into the **same** retrieval pipeline so you can compare backends back-to-back.

**What you'll build (one paragraph).** A pod that (a) clones the [public NRP docs](https://gitlab.nrp-nautilus.io/prp/nrp-site) Markdown, (b) chunks + embeds every page on CPU with `sentence-transformers/all-MiniLM-L6-v2`, (c) writes the vectors to the managed [Milvus](https://nrp.ai/documentation/userdocs/ai/vector-database/) cluster at `milvus.nrp-nautilus.io:50051`, and (d) answers questions by retrieving the top-4 chunks and sending them to an LLM with the system prompt *"answer only from this context — if it isn't there, say so"*. Then we swap the LLM and re-run.

**Cluster prerequisites — what makes this exercise work.** The pod mounts **two Secrets** from `nrp-training-k8s` via `secretKeyRef`:

| Secret | Keys | Source |
|---|---|---|
| `nrp-training-milvus-credentials` | `host`, `port`, `username`, `password`, `secure`, `database` | Issued from [https://nrp.ai/milvus](https://nrp.ai/milvus); pre-loaded for the workshop |
| `nrp-llm-token` | `OPENAI_API_BASE`, `OPENAI_API_KEY` | Bearer token from [https://nrp.ai/llmtoken](https://nrp.ai/llmtoken); pre-loaded for the workshop |

Both Secrets exist in the namespace already. **If you accidentally `kubectl delete secret` either of them**, the next `tutorial-<username>-vectordb` pod will fail with `CreateContainerConfigError` and this entire section breaks for everyone in the namespace — flag a presenter on Matrix and we will re-apply from a backup. (We keep a copy of both Secrets on the presenter laptop precisely for this case.)

#### Stage 1 — Bring up the RAG pod

[`yamls/milvus-rag.yaml`](yamls/milvus-rag.yaml) requests **1 NVIDIA A10**, mounts both Secrets above, then on first boot installs `pymilvus`, `sentence-transformers`, `langchain-text-splitters`, the OpenAI client, **and** [Ollama](https://ollama.com) — pulling the `mistral` 7B model (~4 GB) in the background as a fully offline LLM backend.

```bash
kubectl apply  -n nrp-training-k8s -f yamls/milvus-rag.yaml
kubectl get pod -n nrp-training-k8s tutorial-<username>-vectordb -w
```

The pod reaches `Running` in ~15 seconds, but the bootstrap (apt → pip → ollama install → mistral pull) takes another **3–5 minutes**. Watch progress with:

```bash
kubectl exec -n nrp-training-k8s tutorial-<username>-vectordb -- bash -c '
  python3 -c "import pymilvus, sentence_transformers, openai" 2>&1 | tail -1
  curl -fsS http://localhost:11434/api/tags 2>/dev/null | python3 -m json.tool
'
```

When `models` lists `mistral:latest`, both backends are ready.

#### Stage 2 — Build the index

Open a shell in the pod and pull the script:

```bash
kubectl exec -it -n nrp-training-k8s tutorial-<username>-vectordb -- bash
cd /scratch
curl -fsSLO https://raw.githubusercontent.com/nrp-nautilus/7nrp/main/2_ai_llm_inference_on_nrp/yamls/nrp_docs_rag.py
```

[`yamls/nrp_docs_rag.py`](yamls/nrp_docs_rag.py) is one ~290-line script — read it; nothing magic. Build the corpus once with `--reindex` (sparse-clones nrp-site, chunks 137 markdown files into ~960 pieces, embeds + uploads to Milvus, then runs the built-in question set against whatever LLM `OPENAI_API_BASE` points at):

```bash
python3 nrp_docs_rag.py --reindex
```

Real-run timing: clone 1 s, chunking <1 s, embedding 962 chunks ~25 s on the A10, Milvus inserts ~3 s, six built-in questions ~30 s. **The collection persists across runs** — subsequent invocations skip stages 1–3 and answer in seconds.

#### Stage 3 — Ask the **managed NRP LLM**

The pod inherits `OPENAI_API_BASE=https://ellm.nrp-nautilus.io/v1` and the workshop bearer token from the `nrp-llm-token` Secret, so by default the script's LLM client points at the **managed** endpoint and uses `gemma`. Ask a single question:

```bash
python3 nrp_docs_rag.py --only-ask \
  --ask "What does the cluster do if a pod has no CPU or memory limits?"
```

**Real output (managed gemma):**

```
[5/5] Asking 1 questions
      LLM: model=gemma base=https://ellm.nrp-nautilus.io/v1/

==============================================================================
Q: What does the cluster do if a pod has no CPU or memory limits?
------------------------------------------------------------------------------
The provided context does not state what the cluster does if a pod has no CPU or memory limits.

Sources used:
  - https://nrp.ai/documentation/userdocs/start/using-nautilus   (score=0.684)
  - https://nrp.ai/documentation/userdocs/storage/local           (score=0.641)
  - https://nrp.ai/documentation/userdocs/tutorial/debugging      (score=0.636)
  - https://nrp.ai/documentation/userdocs/start/policies          (score=0.623)
```

Note: the answer is *not in the retrieved chunks* (the docs describe Gatekeeper rejecting pods at admission time, not "what happens" — phrased differently from the question) so the model declines. **This is the desired behavior** for a docs-grounded assistant.

Try one whose answer **is** in the corpus:

```bash
python3 nrp_docs_rag.py --only-ask \
  --ask "How do I get a Milvus database password on NRP, and what is the connection endpoint?"
```

```
==============================================================================
Q: How do I get a Milvus database password on NRP, and what is the connection endpoint?
------------------------------------------------------------------------------
To get your Milvus database password, navigate to the Milvus password page
(/milvus) and click the "Get milvus password" button; a link to a secure page
containing your password will be sent to your email.

The Milvus GRPC endpoint is **milvus.nrp-nautilus.io:50051**.

Source: https://nrp.ai/documentation/userdocs/ai/vector-database
```

Crisp, accurate, with a single source citation — exactly what we want.

#### Stage 4 — Ask the **local Ollama** with the same context

Now point the **same script** at the in-pod Ollama server. Ollama exposes an OpenAI-compatible REST API on `http://localhost:11434/v1`, so the only change is three env vars:

```bash
OPENAI_API_BASE=http://localhost:11434/v1 \
OPENAI_API_KEY=ollama \
RAG_MODEL=mistral \
python3 nrp_docs_rag.py --only-ask \
  --ask "What does the cluster do if a pod has no CPU or memory limits?"
```

**Real output (local mistral 7B Q4):**

```
[5/5] Asking 1 questions
      LLM: model=mistral base=http://localhost:11434/v1/

==============================================================================
Q: What does the cluster do if a pod has no CPU or memory limits?
------------------------------------------------------------------------------
If a pod has no CPU or memory limits, Kubernetes does not guarantee any
specific amount of CPU or memory for that pod. However, it will still schedule
the pod on a node if the node can satisfy the minimum resource requests (if
provided). But if the container exceeds the maximum limit (if set), it will
be killed (OOMKilled - Out of Memory) or throttled (slows down execution),
as mentioned in the context.

Source: https://nrp.ai/documentation/userdocs/start/using-nautilus#resource-allocation

Sources used:
  - https://nrp.ai/documentation/userdocs/start/using-nautilus   (score=0.684)
  ... (identical retrieved chunks as Stage 3)
```

#### Stage 5 — Compare

The retrieved chunks are **byte-for-byte identical** between Stage 3 and Stage 4 (same Milvus index, same `qwen3-style` query embedding) — but the answers differ sharply:

| | Managed `gemma` (NRP) | Local `mistral` (in-pod Ollama) |
|---|---|---|
| Question 1 (unanswerable) | Refuses, says context lacks it | Confabulates — invents OOMKilled / throttling rules from general knowledge, ignores the system prompt |
| Question 2 (answerable) | One paragraph, single citation | Answers correctly, sometimes adds extra Milvus.io links from training data |
| Latency to first token | ~1.5 s, no GPU billed to you | ~6 s, your A10 |
| Token cost | Workshop bearer token (free for now; rate-limited later) | Free as long as your pod runs |
| Privacy | Request leaves your namespace | Request never leaves your pod |

Pick one or the other based on the deployment context — **same code, same retriever, same prompt, different inference backend.** This portability is the entire point.

> 💡 **Why mistral confabulates here.** Smaller models are weaker at honoring the *"only answer from context"* system prompt; they fall back on parametric knowledge. Higher-capacity local models (`llama3.1:70b`, `qwen2.5:32b`, etc.) generally do this better at the cost of more GPU memory. Try `ollama pull llama3.1:8b` and re-run — the comparison gets less dramatic.

#### Stage 6 (optional) — Run from JupyterHub instead of a GPU pod

If you only want to demo the workflow and skip the GPU pod entirely, the JupyterHub server pod already has `OPENAI_API_BASE` / `OPENAI_API_KEY` injected — only the Milvus creds are missing. Build a venv with `--system-site-packages` so it **inherits** the hub's pre-built `torch`/`torchvision`/`transformers` (a plain `python3 -m venv` would force pip to pull a new `torch`, which can mismatch the resident `torchvision` and break `sentence-transformers`):

```bash
python3 -m venv --system-site-packages ~/.venv-rag
source ~/.venv-rag/bin/activate
pip install -q pymilvus sentence-transformers langchain-text-splitters openai

# Milvus creds — paste yours, or copy from kubectl get secret -n nrp-training-k8s
# nrp-training-milvus-credentials -o jsonpath in another terminal.
export MILVUS_HOST=milvus.nrp-nautilus.io MILVUS_PORT=50051 MILVUS_SECURE=true \
       MILVUS_USER=<your-milvus-user> MILVUS_PASSWORD=<your-milvus-password> \
       MILVUS_DB_NAME=<your-milvus-db>

python nrp_docs_rag.py --only-ask --ask "How do I get a Milvus database password on NRP?"
```

If you have not yet built the index (i.e. nobody in the namespace has ever run Stage 2), add `--reindex` once. Embeddings then run on the hub's CPU — slower than Stage 2 (~3 minutes for 962 chunks vs ~25 s on the A10) but completely GPU-free. The Ollama half of Stages 4–5 is unavailable here (the hub has no GPU); use Stage 4 only inside the GPU pod.

#### Cleanup

```bash
kubectl delete -n nrp-training-k8s -f yamls/milvus-rag.yaml
```

The Milvus collection (`nrp_docs_rag`) survives in the managed DB after the pod is gone — the next `tutorial-<username>-vectordb` pod will reuse it without re-indexing unless you pass `--reindex`. To wipe it, pass `--reindex` once or use Attu / `pymilvus` to drop the collection.

---

## 7. Cleanup

```bash
kubectl delete -n nrp-training-k8s -f yamls/pytorch-training.yaml --ignore-not-found
kubectl delete -n nrp-training-k8s -f yamls/tgi-inference.yaml    --ignore-not-found
kubectl delete -n nrp-training-k8s -f yamls/milvus-rag.yaml       --ignore-not-found
```

Stop any `kubectl port-forward` processes, exit the opencode session, and you're done.

---

**Need help?** [Full docs](https://nrp.ai/documentation/) · [Matrix chat](https://nrp.ai/contact/) · [LLM token](https://nrp.ai/llmtoken) · [Milvus password](https://nrp.ai/milvus) · [Models](https://nrp.ai/documentation/userdocs/ai/llm-managed/models/) · [Client configs](https://nrp.ai/documentation/userdocs/ai/llm-managed/client-configs/)
