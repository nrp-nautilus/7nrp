<div align="center">

# 7NRP Tutorials

[![Seventh National Research Platform Workshop](https://img.shields.io/badge/7NRP-Seventh%20National%20Research%20Platform%20Workshop-00529B?style=for-the-badge)](https://na.eventscloud.com/website/91919/)

[![May 5–7 2026 · La Jolla CA](https://img.shields.io/badge/May%205%E2%80%937,%202026-La%20Jolla%2C%20California-0091CD?style=flat-square)](https://na.eventscloud.com/website/91919/)
[![nrp.ai](https://img.shields.io/badge/nrp.ai-Documentation-0FB1A8?style=flat-square)](https://nrp.ai/)
[![Tutorial JupyterHub](https://img.shields.io/badge/Tutorial-JupyterHub-F37726?style=flat-square&logo=jupyter&logoColor=white)](https://training.nrp-nautilus.io/)
[![Matrix](https://img.shields.io/badge/Matrix-Live%20Help-0A2540?style=flat-square&logo=matrix&logoColor=white)](https://nrp.ai/contact/)

</div>

---

Three independent hands-on tutorials for the National Research Platform. Each tutorial lives in its own directory with a single main `.md` file and a `yamls/` subdirectory of manifests it references.

**Presenters**
- Mahidhar Tatineni — San Diego Supercomputer Center, UCSD
- Mohammad Sada — San Diego Supercomputer Center, UCSD
- Daniel Diaz — San Diego Supercomputer Center, UCSD

## Quick start

Two ways to follow along during the workshop:

**Option 1 — Training JupyterHub (recommended, zero install).** The workshop hub at [training.nrp-nautilus.io](https://training.nrp-nautilus.io/) is pre-configured: every spawned JupyterLab pod already has `kubectl` installed and a kubeconfig wired up to the same identity, so you can open a terminal and run `kubectl` immediately. Click below to clone this repo straight into your JupyterLab session:

[![Launch 7NRP Tutorial Workspace](https://img.shields.io/badge/Launch-7NRP%20Tutorial%20Workspace%20%E2%86%92-00529B?style=for-the-badge&logo=jupyter&logoColor=white)](https://training.nrp-nautilus.io/hub/user-redirect/git-pull?repo=https%3A%2F%2Fgithub.com%2Fnrp-nautilus%2F7nrp&branch=main&urlpath=lab%2Ftree%2F7nrp%2F)

**Option 2 — kubectl on your laptop.** Install `kubectl` (Linux / macOS / Windows) and use the ready kubeconfig at [`files/nrp-training.kubeconfig`](files/nrp-training.kubeconfig). It carries the `jupyterhub-sa` service-account token, cluster CA, and `nrp-training-k8s` as the default namespace; the embedded token is valid for the duration of 7NRP, through end-of-day Thursday, May 7, 2026. Step-by-step instructions live in [Tutorial 1 → Interacting with NRP](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md#interacting-with-nrp).

**Conventions**
- Tutorials 1 and 2 share the **`nrp-training-k8s`** namespace. It already exists for the workshop; if you need to recreate it later: `kubectl create namespace nrp-training-k8s`.
- Tutorial 3 uses pre-created per-participant namespaces (**`nrp-training-000`** … **`nrp-training-099`**).
- Replace **`<username>`** in any YAML or command with your NRP or GitHub username to avoid name collisions.
- Every workload must declare CPU and memory `requests` *and* `limits` — a cluster-wide Gatekeeper policy rejects pods without them.

## Tutorials

The three tutorials are independent — pick the one that matches what you want to learn, or work through them in order. Click anywhere on a card (number, title, path, or technology badge) to open the tutorial.

### [![1](https://img.shields.io/badge/1-00529B?style=flat-square) NRP and Kubernetes for Education and Research](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md)

[`1_nrp_kubernetes_education_research/`](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) &nbsp; [![Slides](https://img.shields.io/badge/Slides-NRP%20Overview%20%26%20Jupyter-00529B?style=flat-square&logo=microsoftpowerpoint&logoColor=white)](1_nrp_kubernetes_education_research/NRP-Overview-Jupyter.pptx) &nbsp; [![kubectl](https://img.shields.io/badge/kubectl-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) [![Ingress](https://img.shields.io/badge/Ingress-haproxy-0A2540?style=flat-square)](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) [![NVIDIA GPU](https://img.shields.io/badge/NVIDIA%20GPU-76B900?style=flat-square&logo=nvidia&logoColor=white)](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md)

**Part 1 — intro slides:** [`NRP-Overview-Jupyter.pptx`](1_nrp_kubernetes_education_research/NRP-Overview-Jupyter.pptx) (NRP overview + JupyterHub orientation, presented before the hands-on).
**Part 1 — hands-on:** the [markdown tutorial](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) walks through every core Kubernetes object researchers and educators need: pods, PersistentVolumeClaims, multi-container (sidecar) pods, ConfigMaps + Secrets, Deployments, Batch Jobs, `kubectl cp / port-forward / patch`, HTTPS-fronted Services via Ingress, scheduling primitives (taints, tolerations, labels, affinity) for the workshop's reserved GPU pool, and finally a GPU pod.

### [![2](https://img.shields.io/badge/2-0091CD?style=flat-square) Using AI and LLM Inference on NRP (Jupyter AI, Agents, RAG)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md)

[`2_ai_llm_inference_on_nrp/`](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) &nbsp; [![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![Jupyter AI](https://img.shields.io/badge/Jupyter%20AI-minimax--m2-F37726?style=flat-square&logo=jupyter&logoColor=white)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![MPI](https://img.shields.io/badge/MPI%20%2B%20Horovod-0A2540?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![Ollama](https://img.shields.io/badge/Ollama-0A2540?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![Milvus](https://img.shields.io/badge/Milvus-00A1EA?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![Qualcomm](https://img.shields.io/badge/Qualcomm%20Cloud%20AI%20100-3253DC?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md)

Run AI and LLM inference and training workloads on NRP: PyTorch GPU sanity check, Text Generation Inference, Ollama-based RAG, a Helm-deployed H2O LLM service, distributed TensorFlow training across multiple GPUs (MPI + Horovod), Qualcomm Cloud AI 100 Ultra inference (via JupyterHub profile and via `kubectl`), and a multi-tenant Milvus RAG pipeline that runs the LLM step on either NVIDIA or Qualcomm.

### [![3](https://img.shields.io/badge/3-0FB1A8?style=flat-square) Setting Up Custom JupyterHubs for Classroom and Research](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md)

[`3_custom_jupyterhubs_classroom_research/`](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md) &nbsp; [![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat-square&logo=helm&logoColor=white)](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md) [![JupyterHub](https://img.shields.io/badge/JupyterHub-F37726?style=flat-square&logo=jupyter&logoColor=white)](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md) [![GitLab CI](https://img.shields.io/badge/GitLab%20CI-FC6D26?style=flat-square&logo=gitlab&logoColor=white)](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md)

Deploy your own JupyterHub for a course or research group: install Helm, add the JupyterHub Helm chart, configure Ingress, set up multiple image profiles and per-profile resource limits, attach shared storage, and learn how to build custom container images via NRP GitLab CI/CD.

## Help & support

| | |
|---|---|
| 📚 [User documentation](https://nrp.ai/documentation/userdocs/start/getting-started/) | Getting started on NRP |
| 💬 [Matrix chat](https://nrp.ai/contact/) | Live support during the workshop |
| 🗓️ [7NRP event site](https://na.eventscloud.com/website/91919/) | Schedule, venue, registration |
| 🚀 [Tutorial hub](https://training.nrp-nautilus.io/) | Workshop JupyterHub instance |

---

<div align="center">
<sub>National Research Platform · San Diego Supercomputer Center · UC San Diego</sub>
</div>
