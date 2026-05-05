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

## 📋 Pre-training survey — please take 2 minutes before we start

<a href="images/pre-training-survey-qr.png"><img src="images/pre-training-survey-qr.png" alt="Pre-training survey QR code" width="180" align="right"></a>

Scan the QR on the right (or open it from [`https://ucsantacruz.co1.qualtrics.com/jfe/form/SV_3wQP0UrsPXy3nMO?Q_CHL=qr`](https://ucsantacruz.co1.qualtrics.com/jfe/form/SV_3wQP0UrsPXy3nMO?Q_CHL=qr)) to take the **pre-training survey**. It's a quick set of questions about your prior Kubernetes / NRP / AI experience and what you hope to get out of the workshop.

Your answers let us measure how much each tutorial actually moves the needle — comparing pre-training and post-training responses, plus aggregated session telemetry, is how we study the **efficacy of our training methods** and decide what to keep, cut, or rework for future NRP workshops. The data is collected in aggregate; the more responses we get, the better the next cohort's experience will be.

<br clear="right">


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

[`1_nrp_kubernetes_education_research/`](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) &nbsp; [![Slides](https://img.shields.io/badge/Slides-NRP%20Overview%20%26%20Jupyter%20(PDF)-00529B?style=flat-square&logo=adobeacrobatreader&logoColor=white)](1_nrp_kubernetes_education_research/NRP-Overview-Jupyter.pdf) &nbsp; [![kubectl](https://img.shields.io/badge/kubectl-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) [![Ingress](https://img.shields.io/badge/Ingress-haproxy-0A2540?style=flat-square)](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) [![NVIDIA GPU](https://img.shields.io/badge/NVIDIA%20GPU-76B900?style=flat-square&logo=nvidia&logoColor=white)](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md)

**Part 1 — intro slides:** [`NRP-Overview-Jupyter.pdf`](1_nrp_kubernetes_education_research/NRP-Overview-Jupyter.pdf) (NRP overview + JupyterHub orientation, presented before the hands-on).
**Part 1 — hands-on:** the [markdown tutorial](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) walks through every core Kubernetes object researchers and educators need: pods, PersistentVolumeClaims, multi-container (sidecar) pods, ConfigMaps + Secrets, Deployments, Batch Jobs, `kubectl cp / port-forward / patch`, HTTPS-fronted Services via Ingress, scheduling primitives (taints, tolerations, labels, affinity) for the workshop's reserved GPU pool, and finally a GPU pod.

### [![2](https://img.shields.io/badge/2-0091CD?style=flat-square) Using AI and LLM Inference on NRP (Jupyter AI, Agents, RAG)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md)

[`2_ai_llm_inference_on_nrp/`](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) &nbsp; [![Managed LLM](https://img.shields.io/badge/Managed%20LLM-ellm.nrp--nautilus.io-0FB1A8?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![Jupyter AI](https://img.shields.io/badge/Jupyter%20AI-minimax--m2-F37726?style=flat-square&logo=jupyter&logoColor=white)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![opencode](https://img.shields.io/badge/opencode-Agentic%20CLI-0A2540?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat-square&logo=pytorch&logoColor=white)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![TGI](https://img.shields.io/badge/TGI-Hugging%20Face-FFD21E?style=flat-square&logo=huggingface&logoColor=black)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) [![Milvus](https://img.shields.io/badge/Milvus-00A1EA?style=flat-square)](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md)

Use NRP's GPU-backed managed LLM service: hit the OpenAI-compatible endpoint from Jupyter AI, `curl`, and Python (`openai` SDK); drive an agentic coding TUI (`opencode`) against it to generate a working chess game; then bring up your own GPU pod for a PyTorch sanity check, a TGI inference server, and a Milvus RAG pipeline answering questions over the NRP documentation.

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
