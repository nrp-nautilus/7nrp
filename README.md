# Hands-on Kubernetes, AI/ML, and Custom JupyterHub on the National Research Platform

A three-part tutorial covering how to use the National Research Platform (NRP) — from `kubectl` basics to running LLM inference on NVIDIA and Qualcomm accelerators, to deploying your own JupyterHub via Helm.

**Presenters:**
- Mahidhar Tatineni, San Diego Supercomputer Center, UCSD
- Mohammad Sada, San Diego Supercomputer Center, UCSD
- Daniel Diaz, San Diego Supercomputer Center, UCSD

## Quick Start

To automatically clone this repository into your pre-authenticated JupyterLab environment on NRP, click the following direct link:

**[Launch 7NRP Tutorial Workspace](https://training.nrp-nautilus.io/hub/user-redirect/git-pull?repo=https%3A%2F%2Fgithub.com%2Fnrp-nautilus%2F7nrp&branch=main&urlpath=lab%2Ftree%2F7nrp%2F)**

**Conventions:** Hands-on examples in Parts 1 and 2 use the **`nrp-training-k8s`** namespace; create it with `kubectl create namespace nrp-training-k8s` if it does not exist. Part 3 uses pre-created per-participant namespaces. In any YAML or command, replace **`<username>`** with your NRP or GitHub username to avoid name collisions.

## Agenda

1. [Kubernetes on NRP](part_1_kubernetes/part_1_kubernetes.md) — NRP and Nautilus overview, hosted JupyterHub, `kubectl` basics, GPU pods, MPI jobs.
2. [AI, ML, and LLM Workloads on NRP](part_2_ai_ml_llm/part_2_ai_ml_llm.md) — PyTorch sanity check, Text Generation Inference, Ollama RAG, Helm-based H2O LLM service, Qualcomm Cloud AI 100 Ultra inference (JupyterHub + kubectl), and multi-tenant Milvus RAG.
3. [Custom Hosted JupyterHubs and Helm](part_3_custom_jupyterhub_helm/part_3_custom_jupyterhub_helm.md) — Helm install, JupyterHub Helm chart, Ingress, profiles, shared storage, GitLab CI image builds.

Each part directory contains:
- A single main `.md` with all the steps to follow.
- A `yamls/` subdirectory with the manifests referenced in that part.

## Help and support

- [Get started with NRP](https://nrp.ai/documentation/userdocs/start/getting-started/)
- [Join NRP's Matrix chat](https://nrp.ai/contact/) for hands-on support during the tutorial.
