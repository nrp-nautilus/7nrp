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

Click to clone this repo into your pre-authenticated JupyterLab on the workshop hub:

[![Launch 7NRP Tutorial Workspace](https://img.shields.io/badge/Launch-7NRP%20Tutorial%20Workspace%20%E2%86%92-00529B?style=for-the-badge&logo=jupyter&logoColor=white)](https://training.nrp-nautilus.io/hub/user-redirect/git-pull?repo=https%3A%2F%2Fgithub.com%2Fnrp-nautilus%2F7nrp&branch=main&urlpath=lab%2Ftree%2F7nrp%2F)

**Conventions**
- Tutorials 1 and 2 share the **`nrp-training-k8s`** namespace. It already exists for the workshop; if you need to recreate it later: `kubectl create namespace nrp-training-k8s`.
- Tutorial 3 uses pre-created per-participant namespaces (**`nrp-training-000`** … **`nrp-training-099`**).
- Replace **`<username>`** in any YAML or command with your NRP or GitHub username to avoid name collisions.
- Every workload must declare CPU and memory `requests` *and* `limits` — a cluster-wide Gatekeeper policy rejects pods without them.

## Tutorials

The three tutorials are independent — pick the one that matches what you want to learn, or work through them in order.

### ![1](https://img.shields.io/badge/1-00529B?style=flat-square) NRP and Kubernetes for Education and Research

[`1_nrp_kubernetes_education_research/`](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md) &nbsp; ![kubectl](https://img.shields.io/badge/kubectl-326CE5?style=flat-square&logo=kubernetes&logoColor=white) ![GPU](https://img.shields.io/badge/NVIDIA%20GPU-76B900?style=flat-square&logo=nvidia&logoColor=white) ![MPI](https://img.shields.io/badge/MPI-0A2540?style=flat-square)

Get oriented on the Nautilus cluster, log in to the hosted JupyterHub, and learn `kubectl` basics through hands-on pod creation, GPU pods, and MPI jobs (MPI-pi and a TensorFlow + Horovod benchmark).

### ![2](https://img.shields.io/badge/2-0091CD?style=flat-square) Using AI and LLM Inference on NRP (Jupyter AI, Agents, RAG)

[`2_ai_llm_inference_on_nrp/`](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md) &nbsp; ![PyTorch](https://img.shields.io/badge/PyTorch-EE4C2C?style=flat-square&logo=pytorch&logoColor=white) ![Ollama](https://img.shields.io/badge/Ollama-0A2540?style=flat-square) ![Milvus](https://img.shields.io/badge/Milvus-00A1EA?style=flat-square) ![Qualcomm](https://img.shields.io/badge/Qualcomm%20Cloud%20AI%20100-3253DC?style=flat-square)

Run AI and LLM inference workloads on NRP: PyTorch GPU sanity check, Text Generation Inference, Ollama-based RAG, a Helm-deployed H2O LLM service, Qualcomm Cloud AI 100 Ultra inference (via JupyterHub profile and via `kubectl`), and a multi-tenant Milvus RAG pipeline that runs the LLM step on either NVIDIA or Qualcomm.

### ![3](https://img.shields.io/badge/3-0FB1A8?style=flat-square) Setting Up Custom JupyterHubs for Classroom and Research

[`3_custom_jupyterhubs_classroom_research/`](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md) &nbsp; ![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat-square&logo=helm&logoColor=white) ![JupyterHub](https://img.shields.io/badge/JupyterHub-F37726?style=flat-square&logo=jupyter&logoColor=white) ![GitLab%20CI](https://img.shields.io/badge/GitLab%20CI-FC6D26?style=flat-square&logo=gitlab&logoColor=white)

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
