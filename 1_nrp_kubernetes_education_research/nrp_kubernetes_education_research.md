# NRP and Kubernetes for Education and Research

This tutorial covers using the National Research Platform (NRP) Kubernetes cluster for education and research: logging in to the hosted JupyterHub, opening a terminal that talks to the cluster, exploring resources, and creating Kubernetes objects (pods, GPU pods, MPI jobs).

**Conventions:** Hands-on examples use the **`nrp-training-k8s`** namespace; create it with `kubectl create namespace nrp-training-k8s` if it does not already exist. In any YAML or command, replace **`<username>`** with your NRP or GitHub username to avoid name collisions.

YAMLs referenced in this tutorial live in this directory's [`yamls/`](yamls) folder.

> 📘 **Docs:** [Getting started](https://nrp.ai/documentation/userdocs/start/getting-started/) · [Kubernetes basics](https://nrp.ai/documentation/userdocs/tutorial/basic/) · [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/) · [Run jobs](https://nrp.ai/documentation/userdocs/running/jobs/) · [Storage](https://nrp.ai/documentation/userdocs/storage/intro/) · [JupyterHub](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/) · [Live resources](https://nrp.ai/viz/resources/)

---

## Introduction to the National Research Platform (NRP)

The National Research Platform (NRP) is a partnership of more than 50 institutions, led by researchers and cyberinfrastructure professionals at UC San Diego, University of Nebraska-Lincoln, and the Massachusetts Green High Performance Computing Center (MGHPCC). The NRP provides an open, nationally distributed cyberinfrastructure built on a Kubernetes cluster named **Nautilus**.

Nautilus pools heterogeneous hardware components — spanning compute, storage, and specialized accelerators like NVIDIA GPUs and Qualcomm Cloud AI devices — from contributing partners into a unified computing framework. Researchers access these resources through namespaces, allocating storage, running persistent applications, or executing temporary batch jobs.

### Available Compute Resources

Nautilus features a wide variety of computational resources:
- Standard x86 CPUs and high-memory CPU nodes.
- Diverse **NVIDIA GPUs** (e.g., A10, A100, RTX 3090/4090, H100) accessible for demanding parallel computing tasks and machine learning.
- Advanced hardware accelerators like **Qualcomm Cloud AI 100 Ultra SoCs** natively mapped as standard Kubernetes resources via Device Plugins.

### Storage and Namespaces

By default, your work executes within Kubernetes **Namespaces**. These virtual partitions securely isolate compute workloads and data allocation models across distinct projects. A typical workload utilizes **Persistent Volume Claims (PVCs)** built mostly upon Ceph instances distributed globally. This mechanism allows stateful data generation mapped safely against node eviction policies.

### Scale

- **500+ nodes**
- **1500+ GPUs**
- **50+ FPGAs**

### Capabilities

- **Storage:** CephFS, CVMFS, S3
- **Monitoring:** PerfSONAR, traceroute, Prometheus
- **Compute and data tools:** JupyterHub, WebODM, GitLab, Nextcloud, Overleaf
- **Collaboration tools:** Jitsi, EtherPad, HedgeDoc, Syncthing

### Operational history

The Nautilus cluster has been in continuous operation for **6 years**. Its control plane manages worker nodes that run pods and provide the Kubernetes runtime environment.

---

## Hosted JupyterHub at a glance

NRP runs a [hosted JupyterHub](https://jupyterhub-west.nrp-nautilus.io) you can use with your institutional credentials (CILogon). After logging in, choose the hardware profile for your instance and start running notebooks — no Kubernetes required to begin.

Your home directory (`/home/jovyan`) is a persistent volume, **50GB** by default; don't fill it up or your next Jupyter session may not start. You can request more space or use [CephFS](https://nrp.ai/documentation/userdocs/storage/ceph) for larger or shared workloads. The server will shut down about **1 hour** after your browser disconnects, so keep a tab open or use a stable connection if you need long-running work.

For available images and custom setups, see the [scientific images](https://nrp.ai/documentation/userdocs/running/sci-img/) guide and [TensorFlow with Jupyter](https://nrp.ai/documentation/userdocs/jupyter/jupyter-pod/). More detail: [JupyterHub service](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/).

**Hands-on:** Open [JupyterHub](https://jupyterhub-west.nrp-nautilus.io) (or [training JupyterHub](https://training.nrp-nautilus.io) for the tutorial), log in with CILogon, and spawn an instance with your chosen profile to start using the platform.

---

## Interacting with NRP

The majority of NRP users interact with the cluster using one of three methods:

- via **Kubernetes**: directly submit and manage containerized workloads (services and batch jobs) using Kubernetes APIs and tools like `kubectl`.
- via the **Coder** service: launch a browser-based VS Code environment connected to cluster resources for interactive development and execution.
- via the NRP-deployed **JupyterHub**: start a JupyterLab notebook server on the cluster for interactive analysis, prototyping, and teaching workflows.

In this tutorial we focus on `kubectl`. You have **two ways** to run it for the workshop — pick whichever fits how you like to work:

### Option 1 — Use the training JupyterHub (zero install, recommended)

The workshop hub at [https://training.nrp-nautilus.io](https://training.nrp-nautilus.io) is pre-configured: every spawned JupyterLab pod has `kubectl` installed and a kubeconfig wired to the same `jupyterhub-sa` identity. Open a terminal in JupyterLab and start running `kubectl` immediately — no install, no kubeconfig to manage. **Recommended for the workshop.**

### Option 2 — kubectl on your laptop

Use this if you prefer your own terminal. The repo ships a ready kubeconfig at [`../files/nrp-training.kubeconfig`](../files/nrp-training.kubeconfig) — it carries the `jupyterhub-sa` service-account token, the cluster CA, and `nrp-training-k8s` as the default namespace. The embedded token is valid for the duration of 7NRP, through end-of-day Thursday, May 7, 2026. **This is how to get the kubeconfig for the tutorial.**

**macOS / Linux — single command (installs `kubectl`, installs `helm`, points at the workshop kubeconfig).** Run this from the `7nrp/` directory of your local checkout:

```bash
OS=$(uname | tr '[:upper:]' '[:lower:]') \
&& ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/') \
&& curl -fsSLo /tmp/kubectl "https://dl.k8s.io/release/v1.33.0/bin/${OS}/${ARCH}/kubectl" \
&& sudo install -m 0755 /tmp/kubectl /usr/local/bin/kubectl \
&& curl -fsSL "https://get.helm.sh/helm-v3.16.4-${OS}-${ARCH}.tar.gz" | tar -xz -C /tmp/ \
&& sudo install -m 0755 "/tmp/${OS}-${ARCH}/helm" /usr/local/bin/helm \
&& rm -rf /tmp/kubectl "/tmp/${OS}-${ARCH}" \
&& export KUBECONFIG="$(pwd)/files/nrp-training.kubeconfig" \
&& kubectl auth whoami && helm version --short
```

`KUBECONFIG` is exported in your current shell only. To make it permanent, append the same `export KUBECONFIG="…/files/nrp-training.kubeconfig"` line to `~/.bashrc` or `~/.zshrc`.

**Windows (PowerShell) — equivalent steps.** Run these one at a time from the `7nrp\` directory of your local checkout:

```powershell
winget install -e --id Kubernetes.kubectl
winget install -e --id Helm.Helm
$env:KUBECONFIG = "$(Get-Location)\files\nrp-training.kubeconfig"
kubectl auth whoami
helm version --short
```

To persist `KUBECONFIG` across sessions, add the `$env:KUBECONFIG = …` line to your PowerShell profile (`$PROFILE`).

> **After the workshop ends:** this kubeconfig stops working. For ongoing NRP Nautilus access, configure `kubelogin` to use your personal **CILogon** identity instead of this temporary service account. Follow the official getting-started guide: <https://nrp.ai/documentation/userdocs/start/getting-started/>.

---

## GPUs on NRP

There are many types of GPU available on NRP. You can view live availability of all resources at [https://nrp.ai/viz/resources/](https://nrp.ai/viz/resources/).

### Hands-on: Explore GPU options on NRP

```bash
# print list of NRP nodes with GPU label
kubectl get nodes -L nvidia.com/gpu.product
```

<details>
  <summary>Click to reveal sample output</summary>

```
NAME                                         STATUS   ROLES            AGE      VERSION    GPU.PRODUCT
aarch64.calit2.optiputer.net                 Ready    <none>           2y234d   v1.33.8
admiralty-ncmir-mm-expanse-7d43bc97a0        Ready    cluster,master   13d
cenic-nrp1.hpc.cpp.edu                       Ready    <none>           17d      v1.33.8    NVIDIA-RTX-A6000
chi-dgx-node01.csuchico.edu                  Ready    <none>           165d     v1.33.8    Tesla-V100-SXM2-16GB
clu-fiona2.ucmerced.edu                      Ready    <none>           122d     v1.33.8    NVIDIA-GeForce-GTX-1080-Ti
```
</details>

---

## Kubernetes basics (quick intro)

Kubernetes is a system for running applications on a cluster by managing **workloads** (things you want to run) and keeping them in the desired state.

Most interactions with Kubernetes involve creating and updating **resources** (objects) described in **YAML**.
- A YAML "manifest" declares the *desired state* (what you want running).
- Kubernetes works continuously to make the cluster match that desired state.

Typical workflow:
1. Write or edit a YAML manifest.
2. Apply it to the cluster (e.g., `kubectl apply -f ...`).
3. Check status and troubleshoot (pods, logs, events).

### Kubernetes workloads

Workloads are the resource types you use to run containers on the cluster.

- **Pod**: the basic unit where your application runs (one or more containers together).
- **Job**: runs work to completion (batch or one-off tasks).
- **Deployment**: manages long-running services and keeps them available (including rolling updates).

Rule of thumb:
- Use a **Job** when the work should finish.
- Use a **Deployment** when the work should keep running.

### Hardware acceleration

Kubernetes requires specialized extensions to manage and assign non-CPU hardware.
- **GPUs in Kubernetes**: workloads can explicitly request NVIDIA GPUs (e.g. `nvidia.com/gpu`) by specifying resource limits in their deployment manifests.
- **Device Plugins**: software components that advertise specialized hardware resources to the Kubernetes scheduler. For example, Qualcomm Cloud AI devices are exposed through a device plugin natively mapping as `qualcomm.com/qaic`.

### Keep in mind

- Pods are **ephemeral**. Once a pod is terminated all data is deleted.
- **Persistent Volume Claims** (PVCs) are used to claim long-term storage.
- Kubernetes nodes are typically not accessed directly by users. Instead, users define their workloads in YAML files and submit them to the cluster using `kubectl`, which can be run from any machine that has it installed (such as a local computer or a JupyterHub terminal).

### Docker and containers

Docker is a tool for building and running **containers**. A container image packages:
- your application code,
- libraries and dependencies,
- enough operating-system files to run consistently.

This makes the environment portable: the same image can run on your laptop, a VM, or on a Kubernetes cluster.

#### Why Docker matters for Kubernetes

Kubernetes runs **container images**. It does not build them. In practice:
- You build a container image (with Docker or another tool).
- Kubernetes pulls that image and runs it as part of your workload.

#### Container registries

A **container registry** stores and distributes container images.
- Public example: Docker Hub.
- Organizations often use private registries for internal images.

NRP note:
- NRP GitLab provides a container registry (public or private depending on repo settings).
- You can push local images to GitLab's registry, or build/publish images using GitLab CI/CD.

---

## Hands-on: kubectl basics and a simple pod

In this section we go through some common `kubectl` commands and create some Kubernetes objects.

YAML files are in this directory's [`yamls/`](yamls) folder. Find `test-pod.yaml` and open it in the editor.

### Creating a simple pod

Edit `yamls/test-pod.yaml` to give the pod a unique **name**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-<username>
spec:
  containers:
  - name: mypod
    image: ubuntu
    resources:
      limits:
        memory: 100Mi
        cpu: 100m
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sh", "-c", "echo 'Hello from NRP!' && sleep 3600"]
```

Launch this pod:

```bash
# navigate to yamls directory and create the pod
cd part_1_kubernetes/yamls/
kubectl create -f test-pod.yaml
```

Check whether you were successful:

```bash
kubectl get pods
# get detailed pod information
kubectl get pod test-pod-<username> -o wide
```

Look at the logs associated with this pod:

```bash
kubectl logs test-pod-<username>
```

View detailed pod information:

```bash
kubectl describe pod test-pod-<username>
```

Execute a command in the pod:

```bash
kubectl exec test-pod-<username> -- echo 'Command executed successfully'
```

Get an interactive shell into the pod:

```bash
kubectl exec -it test-pod-<username> -- /bin/bash
```

Finally, clean up the pod to free resources:

```bash
kubectl delete pod test-pod-<username>
```

---

## Monitoring

We collect many metrics in real time to help users evaluate the performance of their workloads. Dashboards (historical and live) are on Grafana:

- [Grafana dashboards](https://grafana.nrp-nautilus.io/dashboards)
- [Grafana: namespace pod dashboard](https://grafana.nrp-nautilus.io/d/85a562078cdf77779eaa1add43ccec1e/kubernetes-compute-resources-namespace-pods)
- [Grafana: namespace GPU dashboard](https://grafana.nrp-nautilus.io/d/dRG9q0Ymz/k8s-compute-resources-namespace-gpus)

Many clusters have an acceptable use policy (including NRP). The most important thing to keep in mind is that **NRP is a shared resource**. Ensure that any resource you request is used efficiently and release the resource when you are done.

At NRP we aim for the utilization of user pods to have GPU > 40%, CPU 20–200%, RAM 20–150% of requested amount.

Cluster usage policies: [https://nrp.ai/documentation/userdocs/start/policies/](https://nrp.ai/documentation/userdocs/start/policies/).

---

## Hands-on: Basic GPU pod

Examine the contents of `yamls/gpu-pod.yaml`. The following block specifies that we are requesting one GPU for the workflow:

```yaml
    resources:
      limits:
        nvidia.com/gpu: 1
      requests:
        nvidia.com/gpu: 1
```

### Resource types

- **NVIDIA GPUs**: in your pod spec, set `resources.limits` and `resources.requests` with the GPU resource key. For a generic GPU, use `nvidia.com/gpu: <count>`.
- **Qualcomm Cloud AI 100**: use `qualcomm.com/qaic: <count>` in `resources.limits` and `resources.requests`. Each unit corresponds to one SoC. Nautilus has **8 Qualcomm Cloud AI 100 Ultra cards**, each with **4 SoCs**, for **32 devices** total; each device can run LLMs up to roughly 25B parameters.
- **Other special GPUs**: for a specific product (e.g., A100, A10, L40, RTX A6000, V100) use the product-specific resource (e.g., `nvidia.com/a100`, `nvidia.com/rtxa6000`).

Now launch the pod:

```bash
# launch single gpu pod
kubectl create -f gpu-pod.yaml
# check that the pod is created
kubectl get pods
```

Once the pod is in a ready state, exec into it:

```bash
kubectl exec -it <username>-gpu-XXXXX -- /bin/bash
```

Try running `nvidia-smi` from within the pod.

**Important:** This pod will remain indefinitely due to the `sleep infinity` command. Terminate it when you are done:

```bash
kubectl delete pod <username>-gpu-XXXXX
```

<details>
<summary>Click to reveal expected nvidia-smi output</summary>

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09             Driver Version: 580.126.09     CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA GeForce GTX 1080 Ti     On  |   00000000:06:00.0 Off |                  N/A |
| 28%   23C    P8              8W /  250W |       3MiB /  11264MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+
```
</details>

### Useful options

#### Specific GPU type

Sometimes you need a specific GPU type. Request nodes with the hardware you need using node affinity:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nvidia.com/gpu.product
            operator: In
            values:
            - NVIDIA-GeForce-GTX-1080-Ti
```

A list of GPU label values is at [https://nrp.ai/documentation/userdocs/running/gpu-pods/](https://nrp.ai/documentation/userdocs/running/gpu-pods/). You can also list the `GPU.PRODUCT` column with:

```bash
kubectl get nodes -L nvidia.com/gpu.product
```

#### Specific CUDA version

If you need a specific CUDA version, use node affinity similarly:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: nvidia.com/cuda.runtime.major
            operator: In
            values:
            - "12"
          - key: nvidia.com/cuda.runtime.minor
            operator: In
            values:
            - "2"
```

You can list CUDA versions per node:

```bash
kubectl get nodes -L nvidia.com/cuda.driver.major,nvidia.com/cuda.driver.minor,nvidia.com/cuda.runtime.major,nvidia.com/cuda.runtime.minor -l nvidia.com/gpu.product
```

#### Special GPUs

Some GPUs are labeled as special resources on the cluster and cannot be scheduled using `nvidia.com/gpu`. See [GPU pods documentation](https://nrp.ai/documentation/userdocs/running/gpu-pods/) for more.

```yaml
    resources:
      limits:
        nvidia.com/a40: 1
      requests:
        nvidia.com/a40: 1
```

---

## Hands-on: TensorFlow benchmarking (MPI + Horovod)

For this job, one launcher pod creates two GPU worker pods and runs a distributed TensorFlow ResNet-101 benchmark across them using MPI and Horovod.

First, edit `yamls/mpi-tensorflow.yaml` and replace `<username>` with a unique name.

```bash
kubectl create -f mpi-tensorflow.yaml
```

To check progress:

```bash
kubectl get pods
# once the launcher pod is in a running state
kubectl logs -f <username>-mpi-tensorflow-XXXXX-launcher-YYYYY
```

### Questions

- What metrics would you look at for this job?
- Which pods do you expect to see GPU utilization on?
- How is your GPU utilization for this job? How about memory?
- Do the number of CPUs you are requesting seem appropriate?
- Should you adjust any resource request for better efficiency?

---

## Hands-on: MPI-pi

We will use a custom workload called an **mpijob**, created using `yamls/mpi-pi.yaml`. This workload creates one launcher pod and two worker pods. The launcher pod initiates a job to compute π and the worker pods execute it.

First, edit `yamls/mpi-pi.yaml` replacing `<username>` with a unique name.

```bash
kubectl create -f mpi-pi.yaml
```

Check progress:

```bash
kubectl get pods
# once the launcher pod is in a running state
kubectl logs -f <username>-mpi-pi-XXXXX-launcher-YYYYY
```

<details>
  <summary>Click to reveal expected result</summary>

```
# running on worker pods
Rank 1 on host <username>-mpi-pi-4lrz7-worker-1
Workers: 2
Rank 0 on host <username>-mpi-pi-4lrz7-worker-0
pi is approximately 3.1410376000000002
```

or, on fallback:

```
Distributed transport failed; falling back to local launcher-only run for demo reliability...
Workers: 2
Rank 0 on host <username>-mpi-pi-jzrvt-launcher
Rank 1 on host <username>-mpi-pi-jzrvt-launcher
pi is approximately 3.1410376000000002
```
</details>

To clean up:

```bash
kubectl get mpijob
kubectl delete mpijob <username>-mpi-pi-XXXXX
```

### Question

- What metrics would you look at for this job?

---

## End — cleanup

**Please make sure you did not leave any running pods.** Jobs and associated completed pods are OK.

```bash
# delete anything you created in this part
kubectl delete pod test-pod-<username> --ignore-not-found
kubectl delete pod <username>-gpu-XXXXX --ignore-not-found
kubectl delete mpijob <username>-mpi-pi-XXXXX --ignore-not-found
kubectl delete mpijob <username>-mpi-tensorflow-XXXXX --ignore-not-found
```

**Need help?** [Full docs](https://nrp.ai/documentation/) · [Matrix chat](https://nrp.ai/contact/) · [FAQ](https://nrp.ai/documentation/userdocs/start/faq/) · [Policies](https://nrp.ai/documentation/userdocs/start/policies/)

**Related docs:** [Using Nautilus](https://nrp.ai/documentation/userdocs/start/using-nautilus/) · [Kubernetes basics](https://nrp.ai/documentation/userdocs/tutorial/basic/) · [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/) · [Run jobs](https://nrp.ai/documentation/userdocs/running/jobs/) · [Storage](https://nrp.ai/documentation/userdocs/storage/intro/) · [JupyterHub service](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/) · [Live resources](https://nrp.ai/viz/resources/)
