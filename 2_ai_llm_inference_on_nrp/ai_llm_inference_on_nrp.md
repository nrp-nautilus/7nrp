# Using AI and LLM Inference on NRP (Jupyter AI, Agents, RAG)

This tutorial covers running AI and LLM inference workloads on NRP: using JupyterAI inside notebooks, deploying inference models on NVIDIA A10 GPUs and Qualcomm Cloud AI 100 Ultra cards, and building a RAG pipeline with the NRP-managed Milvus vector database.

**Conventions:** Hands-on examples use the **`nrp-training-k8s`** namespace; create it with `kubectl create namespace nrp-training-k8s` if it does not already exist. In any YAML or command, replace **`<username>`** with your NRP or GitHub username to avoid name collisions.

YAMLs referenced here live in this directory's [`yamls/`](yamls) folder.

### NRP LLM API at a glance

NRP serves open-weights LLMs behind an **OpenAI-compatible API** at `https://ellm.nrp-nautilus.io/v1`. You can try them in the browser via [Open WebUI](https://nrp-openwebui.nrp-nautilus.io) or [LibreChat](https://librechat.nrp-nautilus.io), or call the API with a token from the [LLM token page](https://nrp.ai/llmtoken). The [NRP LLM documentation](https://nrp.ai/documentation/userdocs/ai/llm-managed/) lists available models (e.g., glm-v, gpt-oss), context lengths, and tool-calling support.

For **Qualcomm Cloud AI 100** inference, the cluster has 8 cards (32 SoCs). Request `qualcomm.com/qaic` and use the image `gitlab-registry.nrp-nautilus.io/cloud-ai-100/qaic-docker-images:vllm-latest` for vLLM workloads. See [Qualcomm Cloud AI 100](https://nrp.ai/documentation/userdocs/qaic/) for full details.

---

# A. Inference workloads on NVIDIA GPUs

The following sections demonstrate inference workloads running on **NVIDIA GPUs** (e.g., A10). These workflows use standard CUDA-enabled containers and request GPUs via `nvidia.com/gpu` in Kubernetes resource limits.

## A.1 PyTorch GPU sanity check

We start with a **PyTorch GPU sanity pod** (`yamls/pytorch-training.yaml`): it runs `nvidia-smi` and a short CUDA matmul so you can confirm the NVIDIA stack on an A10 node.

```bash
kubectl apply -n nrp-training-k8s -f yamls/pytorch-training.yaml
```

Check status:

```bash
kubectl get pods -n nrp-training-k8s
```

When the pod has run to completion (or while running), check the logs — you should see `nvidia-smi` and `GPU matmul OK`:

```bash
kubectl logs -n nrp-training-k8s tutorial-<username>-gp3
```

Once you are done exploring, delete the pod:

```bash
kubectl delete -n nrp-training-k8s -f yamls/pytorch-training.yaml
```

---

## A.2 Text Generation Inference (TGI)

Start the inference pod:

```bash
kubectl apply -n nrp-training-k8s -f yamls/tgi-inference.yaml
```

Once the pod is running, get interactive access:

```bash
kubectl exec -it -n nrp-training-k8s tutorial-<username>-tgi -- /bin/bash
```

Inside the pod, start a Python interpreter and run:

```python
from huggingface_hub import InferenceClient
client = InferenceClient(model="http://0.0.0.0:80")
for token in client.text_generation("Who made cat videos?", max_new_tokens=24, stream=True):
    print(token)
```

---

## A.3 RAG example with Ollama

Start the pod:

```bash
kubectl apply -n nrp-training-k8s -f yamls/ollama-rag.yaml
```

Watch the logs and wait until the installs and book download complete:

```bash
kubectl logs -n nrp-training-k8s tutorial-<username>-ollama
```

Once the book is downloaded (you'll see `wget` output in the logs), exec into the pod and start the Ollama server, then pull Mistral:

```bash
kubectl exec -it -n nrp-training-k8s tutorial-<username>-ollama -- /bin/bash
cd /scratch
nohup ollama serve &
ollama pull mistral
```

Download our test script and run it:

```bash
wget https://raw.githubusercontent.com/nrp-nautilus/nairr-tutorial/main/scripts/test.py
python3 -i test.py
```

Inside the interactive Python interpreter, run the queries one by one (wait for each result before moving on):

```python
rag.invoke("What do you feed pigeons ?")
rag.invoke("Do tame pigeons have better plumage ?")
rag.invoke("What affects pigeon plumage ?")
```

---

## A.4 LLM as a service: Helm-based H2O deployment

This section deploys H2O GPT via Helm. **Install Helm** so you can run `helm`:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Make sure `~/.local/bin` is on your `PATH`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Or in a Python/notebook cell:

```python
import os
os.environ["PATH"] = os.path.expanduser("~/.local/bin:") + os.environ.get("PATH", "")
```

Clone the [H2O project](https://github.com/h2oai):

```bash
git clone https://github.com/h2oai/h2ogpt.git
```

Copy the values file into the cloned repo and `cd` in:

```bash
cp yamls/h2o-values.yaml h2ogpt/
cd h2ogpt
```

Install the Helm chart from **inside the `h2ogpt` directory**. Use a unique release name (replace `username`):

```bash
helm install h2ogpt-<username> helm/h2ogpt-chart -f h2o-values.yaml
```

Check the pods (`kubectl get pods`, optionally `grep` your username) and wait until they are running. The model will take some time to load into GPU memory.

Port-forward to the pod from a terminal:

```bash
kubectl port-forward h2ogpt-<username>-<hash> 5000:5000 7860:7860
```

While port-forward is running, list models from another shell:

```bash
curl http://localhost:5000/v1/models
```

Or open [http://localhost:5000/v1/models](http://localhost:5000/v1/models) in a browser.

Run a query:

```bash
curl -X POST "http://localhost:5000/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "h2oai/h2o-danube2-1.8b-chat",
    "messages": [
      {"role": "user", "content": "How tall are penguins?"}
    ]
  }'
```

The chart also installs a chat application — open [http://localhost:7860](http://localhost:7860) in a browser to access the chat UI.

You can expose the LLM via an Ingress (carefully edit all fields with your username):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: haproxy
  name: llm-test-<username>
spec:
  rules:
  - host: llm-test-<username>.nrp-nautilus.io
    http:
      paths:
      - backend:
          service:
            name: h2ogpt-<username>-web
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - llm-test-<username>.nrp-nautilus.io
```

When you are done, release the resources:

```bash
helm uninstall h2ogpt-<username>
```

---

## A.5 Cleanup (NVIDIA section)

Delete anything you created so nothing is left running (replace `<username>` / release name as you used):

```bash
# PyTorch training (if still running)
kubectl delete -n nrp-training-k8s -f yamls/pytorch-training.yaml --ignore-not-found

# TGI inference pod
kubectl delete -n nrp-training-k8s -f yamls/tgi-inference.yaml --ignore-not-found

# Ollama RAG pod
kubectl delete -n nrp-training-k8s -f yamls/ollama-rag.yaml --ignore-not-found

# H2O Helm release
helm uninstall h2ogpt-<username>
```

Stop any `kubectl port-forward` processes you started.

---

# B. Inference workloads on Qualcomm Cloud AI 100 Ultra

The following sections demonstrate inference workloads running on **Qualcomm Cloud AI 100 Ultra** accelerators. These devices are exposed in Kubernetes as `qualcomm.com/qaic` and use the SDK container image `ghcr.io/quic/cloud_ai_inference_ubuntu24:1.20.6.0`.

You have two options for running Qualcomm inference:
1. **Via JupyterHub** — select the NRP Cloud AI Inference profile from the spawner.
2. **Via kubectl** — deploy a standalone pod using `yamls/qaic-inference.yaml`.

## B.1 Qualcomm via JupyterHub

The NRP provides access to Qualcomm Cloud AI 100 Ultra devices directly through custom JupyterHub profiles. This launches an isolated environment with the Qualcomm Platform and Apps SDKs pre-installed.

### Launching the Qualcomm profile

1. Navigate to the [NRP Training JupyterHub](https://training.nrp-nautilus.io).
2. To utilize Qualcomm inference accelerators, select the **NRP Cloud AI Inference** option from the Spawner Profile list.
3. In the form, locate the **Qualcomm Cloud AI Devices** picker and select exactly **1** device.
4. Click **Start** to spawn your instance. Your pod will execute as the `jovyan` user (UID 1000) with a dedicated `50Gi` persistent volume mounted.

### Accessing the Qualcomm environment

Once your JupyterLab spins up, open a **Terminal** from the launcher. You'll see a welcome message confirming the SDK versions:

```
==================================
== Qualcomm Cloud AI Containers ==
==================================

Platform SDK version: 1.20.6.0
Apps SDK version: 1.20.6.0
```

### Activating the pre-built vLLM environment

The Qualcomm image bundles an optimized Python virtual environment with the vLLM bindings compiled against the QAIC architecture. Source it:

```bash
source /opt/vllm-env/bin/activate
```

### Running example inference

Built-in offline inference examples are under `/opt/qti-aic/integrations/vllm/`. For **vision-language** (e.g., OpenGVLab/InternVL2_5-1B):

```bash
source /opt/vllm-env/bin/activate
cd /opt/qti-aic/integrations/vllm
python examples/offline_inference/qaic_vision_language.py \
  --image-url https://huggingface.co/datasets/huggingface/documentation-images/resolve/0052a70beed5bf71b92610a43a52df6d286cd5f3/diffusers/rabbit.jpg \
  --question "What's in the image?" \
  -m internvl_chat \
  --num-prompt 1
```

Alternative (KV offload): `python examples/offline_inference/qaic_vision_language_kv_offload.py` with the same `--image-url` and `--question` arguments.

---

## B.2 Qualcomm via kubectl

Deploy a Qualcomm inference pod directly with `kubectl`, similar to the NVIDIA TGI flow above. The pod requests `qualcomm.com/qaic: 1` and runs `sleep infinity`. After the pod is Running, exec in and run inference.

```bash
kubectl apply -n nrp-training-k8s -f yamls/qaic-inference.yaml
```

Check pod status until it is Running:

```bash
kubectl get pods -n nrp-training-k8s
```

Exec into the pod and run the vision-language inference. Set `HOME=/scratch`, activate the vLLM env, and run the example:

```bash
kubectl exec -n nrp-training-k8s tutorial-<username>-qaic-inference -- bash -c '
  cd /scratch
  export HOME=/scratch
  source /opt/vllm-env/bin/activate
  cd /opt/qti-aic/integrations/vllm
  python examples/offline_inference/qaic_vision_language.py \
    --image-url https://huggingface.co/datasets/huggingface/documentation-images/resolve/0052a70beed5bf71b92610a43a52df6d286cd5f3/diffusers/rabbit.jpg \
    --question "What'\''s in the image?" \
    -m internvl_chat \
    --num-prompt 1'
```

### vLLM OpenAI-compatible API server on QAIC

You can run vLLM's OpenAI-compatible API server on the same Qualcomm pod and then call it with `curl` or any OpenAI SDK. The [Qualcomm Cloud AI vLLM guide](https://quic.github.io/cloud-ai-sdk-pages/latest/Getting-Started/Installation/vLLM/vLLM/index.html) documents supported models and QAIC-specific flags (`--device qaic`, `--quantization mxfp6`, `--kv-cache-dtype mxint8`). Use a model known to work on QAIC (e.g., `TinyLlama/TinyLlama-1.1B-Chat-v1.0`). The server takes a few minutes to load and compile the model.

Start the server inside the pod (background; logs go to `/scratch/vllm.log`):

```bash
kubectl exec -n nrp-training-k8s tutorial-<username>-qaic-inference -- bash -c '
  source /opt/vllm-env/bin/activate
  export HOME=/scratch XDG_CACHE_HOME=/scratch QEFF_HOME=/scratch OMP_NUM_THREADS=8
  nohup python3 -m vllm.entrypoints.openai.api_server \
    --host 0.0.0.0 --port 8000 \
    --model TinyLlama/TinyLlama-1.1B-Chat-v1.0 \
    --max-model-len 256 --max-num-seq 16 --max-seq-len-to-capture 128 \
    --device qaic --block-size 32 \
    --quantization mxfp6 --kv-cache-dtype mxint8 \
    > /scratch/vllm.log 2>&1 &
  echo started'
```

Port-forward from your machine (run in a terminal, leave it running):

```bash
kubectl port-forward -n nrp-training-k8s tutorial-<username>-qaic-inference 8000:8000
```

Call the API:

```bash
curl -s http://127.0.0.1:8000/v1/models
curl -s -X POST http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"TinyLlama/TinyLlama-1.1B-Chat-v1.0","messages":[{"role":"user","content":"Say hello."}],"max_tokens":20}'
```

Once done, clean up:

```bash
kubectl delete -n nrp-training-k8s -f yamls/qaic-inference.yaml
```

---

# C. Multi-tenant vector databasing with Milvus for RAG

NRP runs **Milvus** as a managed multi-tenant vector database service. You don't install the operator or create the instance yourself — the gRPC endpoint is `milvus.nrp-nautilus.io:50051`. To get access, use the [Milvus password page](https://nrp.ai/milvus) and click "Get milvus password" — the secure link is emailed to you. You must be in a [namespace/group with Milvus enabled](https://nrp.ai/namespaces). Group names may use letters, numbers, and dashes; any dashes are converted to underscores in the Milvus database name.

From there you can [define collections](https://milvus.io/docs/manage-collections.md), run [vector search](https://milvus.io/docs/single-vector-search.md), or use the [Attu GUI](https://milvus.io/docs/quickstart_with_attu.md). Full reference: [NRP vector database](https://nrp.ai/documentation/userdocs/vector-database/).

## C.1 RAG example using Milvus (NVIDIA backend)

**Prerequisite:** get your Milvus password and ensure your group has Milvus enabled before starting the pod.

In `yamls/milvus-rag.yaml`, **replace `<username>`** so the pod name `tutorial-<username>-vectordb` is unique. Apply:

```bash
kubectl apply -n nrp-training-k8s -f yamls/milvus-rag.yaml
```

Watch the logs and wait until installs are done:

```bash
kubectl logs -n nrp-training-k8s tutorial-<username>-vectordb
```

The pod automatically installs Python dependencies, Ollama, starts the Ollama server, and downloads the Mistral model. Download the example script (example repo: [groundsada/nrp-milvus-example](https://github.com/groundsada/nrp-milvus-example)):

```bash
kubectl exec -it -n nrp-training-k8s tutorial-<username>-vectordb -- /bin/bash
# inside the pod:
cd /scratch
wget https://raw.githubusercontent.com/groundsada/nrp-milvus-example/refs/heads/main/milvus-example.py
```

Once the install is complete, run the example. The script uses the Milvus connection from the secret `nrp-training-milvus-credentials` (host, port, username, password, database). The collection name is `simple_rag_example`. To use a different collection, edit `milvus-example.py` and change `collection_name="simple_rag_example"` (two places).

Run inside the pod:

```bash
cd /scratch
python3 milvus-example.py
```

## C.2 Milvus RAG with the LLM on Qualcomm

You can run the same RAG pipeline with the **LLM on Qualcomm Cloud AI 100 Ultra**: embeddings and Milvus stay on the existing pod; only the generative step uses a vLLM server running on QAIC.

Two ways:

1. **Via JupyterHub** — use the **NRP Cloud AI Inference** profile (B.1): spawn with 1 Qualcomm device, then run the QAIC vLLM server and RAG steps from a terminal there. The Milvus RAG pod (C.1) must be running in the same namespace, and the RAG script must point `OPENAI_API_BASE` at the QAIC server (port-forward or in-cluster service).
2. **Via kubectl** — deploy the QAIC vLLM server pod and service, then run the RAG script from the Milvus RAG pod.

### Deploy the QAIC vLLM server

In `yamls/qaic-vllm-server.yaml` replace `<username>` in the pod name, then apply. The pod runs an OpenAI-compatible vLLM server (TinyLlama) on port 8000; the service `qaic-vllm-server` lets other pods in `nrp-training-k8s` reach it (a namespace exception may be required; see [NRP policies](https://nrp.ai/documentation/userdocs/start/policies/)).

```bash
kubectl apply -n nrp-training-k8s -f yamls/qaic-vllm-server.yaml
```

Check pod status and logs until vLLM is ready:

```bash
kubectl get pods -n nrp-training-k8s -l app=qaic-vllm-server
kubectl logs -n nrp-training-k8s tutorial-<username>-qaic-vllm
```

Run the Milvus RAG script **from the Milvus RAG pod**, pointing the LLM at the QAIC server:

```bash
kubectl exec -it -n nrp-training-k8s tutorial-<username>-vectordb -- bash -c '
  cd /scratch
  wget -q https://raw.githubusercontent.com/nrp-nautilus/nairr-tutorial/main/scripts/milvus_rag_qaic.py
  OPENAI_API_BASE=http://qaic-vllm-server:8000/v1 python3 milvus_rag_qaic.py'
```

The script uses the same Milvus credentials and collection; only the generative step uses the Qualcomm vLLM server.

## C.3 Milvus cleanup

Delete the Milvus RAG pod and, if used, the QAIC vLLM server:

```bash
kubectl delete -n nrp-training-k8s -f yamls/milvus-rag.yaml --ignore-not-found
kubectl delete -n nrp-training-k8s -f yamls/qaic-vllm-server.yaml --ignore-not-found
```


# End — final checklist

- Make sure no GPU/QAIC pods are still running unintentionally — they are a shared resource.
- `helm uninstall` any H2O release you created.
- Stop any `kubectl port-forward` processes you started.

**Join NRP & hands-on support:** [Get started with NRP](https://nrp.ai/documentation/userdocs/start/getting-started/) · [Join NRP's Matrix chat](https://nrp.ai/contact/).

**Related documentation:** [NRP-managed LLMs](https://nrp.ai/documentation/userdocs/ai/llm-managed/) · [Qualcomm Cloud AI 100](https://nrp.ai/documentation/userdocs/qaic/) · [NRP-managed vector database (Milvus)](https://nrp.ai/documentation/userdocs/vector-database/).
