<div align="center">

# 7NRP Tutorials

**Seventh National Research Platform Workshop**
May 5–7, 2026 · La Jolla, California

[Event site](https://na.eventscloud.com/website/91919/) · [nrp.ai](https://nrp.ai/) · [Tutorial JupyterHub](https://training.nrp-nautilus.io/)

</div>

---

Three independent hands-on tutorials for the National Research Platform (NRP). Each tutorial lives in its own directory with a single main `.md` file and a `yamls/` subdirectory containing the manifests it references.

**Presenters**
- Mahidhar Tatineni — San Diego Supercomputer Center, UCSD
- Mohammad Sada — San Diego Supercomputer Center, UCSD
- Daniel Diaz — San Diego Supercomputer Center, UCSD

## Quick start

To clone this repository into your pre-authenticated JupyterLab on the workshop hub, click:

**[Launch 7NRP Tutorial Workspace →](https://training.nrp-nautilus.io/hub/user-redirect/git-pull?repo=https%3A%2F%2Fgithub.com%2Fnrp-nautilus%2F7nrp&branch=main&urlpath=lab%2Ftree%2F7nrp%2F)**

**Conventions**
- Tutorials 1 and 2 share the **`nrp-training-k8s`** namespace. It already exists for the workshop; if you need to recreate it later, `kubectl create namespace nrp-training-k8s`.
- Tutorial 3 uses pre-created per-participant namespaces (`nrp-training-000` … `nrp-training-099`).
- Replace **`<username>`** in any YAML or command with your NRP or GitHub username to avoid name collisions.
- Every workload must declare CPU and memory `requests` *and* `limits` — a cluster-wide Gatekeeper policy rejects pods without them.

## Tutorials

The three tutorials are independent — pick the one that matches what you want to learn, or work through them in order.

### 1 · NRP and Kubernetes for Education and Research
[`1_nrp_kubernetes_education_research/`](1_nrp_kubernetes_education_research/nrp_kubernetes_education_research.md)

Get oriented on the Nautilus cluster, log in to the hosted JupyterHub, and learn `kubectl` basics through hands-on pod creation, GPU pods, and MPI jobs (MPI-pi and a TensorFlow + Horovod benchmark).

### 2 · Using AI and LLM Inference on NRP (Jupyter AI, Agents, RAG)
[`2_ai_llm_inference_on_nrp/`](2_ai_llm_inference_on_nrp/ai_llm_inference_on_nrp.md)

Run AI and LLM inference workloads on NRP: PyTorch GPU sanity check, Text Generation Inference, Ollama-based RAG, a Helm-deployed H2O LLM service, Qualcomm Cloud AI 100 Ultra inference (via JupyterHub profile and via `kubectl`), and a multi-tenant Milvus RAG pipeline that can run with the LLM step on either NVIDIA or Qualcomm.

### 3 · Setting Up Custom JupyterHubs for Classroom and Research
[`3_custom_jupyterhubs_classroom_research/`](3_custom_jupyterhubs_classroom_research/custom_jupyterhubs_classroom_research.md)

Deploy your own JupyterHub for a course or research group: install Helm, add the JupyterHub Helm chart, configure Ingress, set up multiple image profiles and per-profile resource limits, attach shared storage, and learn how to build custom container images via NRP GitLab CI/CD.

## Help & support

- [Get started with NRP](https://nrp.ai/documentation/userdocs/start/getting-started/) — official user docs
- [NRP Matrix chat](https://nrp.ai/contact/) — hands-on support during the workshop
- [7NRP event site](https://na.eventscloud.com/website/91919/) — schedule, venue, registration

---

<sub>Hosted on the National Research Platform · running on the workshop training hub at <a href="https://training.nrp-nautilus.io/">training.nrp-nautilus.io</a></sub>
