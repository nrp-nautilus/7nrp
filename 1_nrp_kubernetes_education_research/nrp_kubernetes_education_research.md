# NRP and Kubernetes for Education and Research

This tutorial covers using the National Research Platform (NRP) Kubernetes cluster for education and research: logging in to the hosted JupyterHub, opening a terminal that talks to the cluster, exploring resources, and creating the Kubernetes objects researchers and educators reach for daily — Pods, PersistentVolumeClaims, multi-container (sidecar) Pods, ConfigMaps + Secrets, Deployments, Batch Jobs, an HTTPS-fronted Service via Ingress, and GPU pods. Along the way you'll learn `kubectl cp` / `port-forward` / `patch`, plus how scheduling primitives (node labels, affinity, taints, tolerations) let you steer pods onto the workshop's reserved GPU pool.

<!--
**Conventions:** Hands-on examples use the **`nrp-training-k8s`** namespace; create it with `kubectl create namespace nrp-training-k8s` if it does not already exist. In any YAML or command, replace **`<username>`** with your NRP or GitHub username to avoid name collisions.
-->

YAMLs referenced in this tutorial live in this directory's [`yamls/`](yamls) folder.

> 📘 **Docs:** [Getting started](https://nrp.ai/documentation/userdocs/start/getting-started/) · [Kubernetes basics](https://nrp.ai/documentation/userdocs/tutorial/basic/) · [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/) · [Run jobs](https://nrp.ai/documentation/userdocs/running/jobs/) · [Storage](https://nrp.ai/documentation/userdocs/storage/intro/) · [JupyterHub](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/) · [Live resources](https://nrp.ai/viz/resources/)

---

## Introduction

The National Research Platform (NRP) is a partnership of 50+ institutions providing an open, nationally distributed cyberinfrastructure built on a Kubernetes cluster named **Nautilus** — 500+ nodes, 1500+ GPUs (NVIDIA A10/A100/H100, Qualcomm Cloud AI 100 Ultra), 50+ FPGAs, in continuous operation for six years. Researchers and educators access it via Kubernetes namespaces, with persistent storage on Ceph and shared services for JupyterHub, GitLab, Coder, and S3.

> **Slide deck:** [`NRP-Overview-Jupyter.pptx`](NRP-Overview-Jupyter.pptx) — the instructor's overview of NRP and JupyterHub. This markdown is the hands-on companion that runs *after* the slides.

---
## Interacting with NRP

![Top Uses](../images/TopUse.png)

The majority of NRP users interact with the cluster using the following three methods.
- **Kubernetes**: Directly submit and manage containerized workloads (services and batch jobs) using Kubernetes APIs and tools like `kubectl`.
- **Coder**: Launch a browser-based VS Code environment connected to cluster resources for interactive development and execution.
- **Jupyterhub**: Start a JupyterLab notebook server on the cluster for interactive analysis, prototyping, and teaching workflows.

---

## Hosted JupyterHub at a glance

NRP runs a hosted JupyterHub at https://jupyterhub-west.nrp-nautilus.io you can use with your institutional credentials (CILogon). After logging in, choose the hardware profile for your instance and start running notebooks — no Kubernetes required to begin.

Your home directory (`/home/jovyan`) is a persistent volume, **5GB** by default; don't fill it up or your next Jupyter session may not start. You can request more space or use [CephFS](https://nrp.ai/documentation/userdocs/storage/ceph) for larger or shared workloads. The server will shut down about **1 hour** after your browser disconnects, so keep a tab open or use a stable connection if you need long-running work.

For available images and custom setups, see the [scientific images](https://nrp.ai/documentation/userdocs/running/sci-img/) guide and [TensorFlow with Jupyter](https://nrp.ai/documentation/userdocs/jupyter/jupyter-pod/). More detail: [JupyterHub service](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/).

>**Hands-on:** Open the [training JupyterHub](https://training.nrp-nautilus.io) for the tutorial), log in with CILogon, and spawn an instance with your chosen profile to start using the platform. Optionally follow the instructions below to install kubectl on your laptop.

---


### Option 1 — Use the training JupyterHub (zero install, recommended)

The workshop hub at [https://training.nrp-nautilus.io](https://training.nrp-nautilus.io) is pre-configured: every spawned JupyterLab pod has `kubectl` installed and a kubeconfig wired to the same `jupyterhub-sa` identity. Open a terminal in JupyterLab and start running `kubectl` immediately — no install, no kubeconfig to manage. **Recommended for the workshop.**

### Option 2 — kubectl on your laptop
<details>
<summary>kubectl on your laptop instructions</summary>


Use this if you prefer your own terminal. The workshop kubeconfig carries the `jupyterhub-sa` service-account token, the cluster CA, and `nrp-training-k8s` as the default namespace. The embedded token is valid for the duration of 7NRP, through end-of-day Thursday, May 7, 2026.

Three universal copy-paste commands — install `kubectl`, install `helm`, then add the workshop kubeconfig. None of them assume you have cloned this repo locally.

**macOS / Linux**

1. Install `kubectl`:

```bash
OS=$(uname | tr '[:upper:]' '[:lower:]') \
&& ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/') \
&& curl -fsSLo /tmp/kubectl "https://dl.k8s.io/release/v1.33.0/bin/${OS}/${ARCH}/kubectl" \
&& sudo install -m 0755 /tmp/kubectl /usr/local/bin/kubectl \
&& rm /tmp/kubectl \
&& kubectl version --client
```

2. Install `helm`:

```bash
OS=$(uname | tr '[:upper:]' '[:lower:]') \
&& ARCH=$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/') \
&& curl -fsSL "https://get.helm.sh/helm-v3.16.4-${OS}-${ARCH}.tar.gz" | tar -xz -C /tmp/ \
&& sudo install -m 0755 "/tmp/${OS}-${ARCH}/helm" /usr/local/bin/helm \
&& rm -rf "/tmp/${OS}-${ARCH}" \
&& helm version --short
```

3. Add the workshop kubeconfig (download into `~/.kube/`, merge into your default config, and switch context):

```bash
mkdir -p ~/.kube \
&& curl -fsSLo ~/.kube/nrp-training.kubeconfig https://raw.githubusercontent.com/nrp-nautilus/7nrp/main/files/nrp-training.kubeconfig \
&& chmod 600 ~/.kube/nrp-training.kubeconfig \
&& KUBECONFIG=~/.kube/config:~/.kube/nrp-training.kubeconfig kubectl config view --flatten > ~/.kube/config.merged \
&& mv ~/.kube/config.merged ~/.kube/config && chmod 600 ~/.kube/config \
&& kubectl config use-context nrp-training-k8s \
&& kubectl auth whoami
```

**Windows (PowerShell)**

1. Install `kubectl`:

```powershell
winget install -e --id Kubernetes.kubectl ; kubectl version --client
```

2. Install `helm`:

```powershell
winget install -e --id Helm.Helm ; helm version --short
```

3. Add the workshop kubeconfig:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.kube" | Out-Null ; `
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/nrp-nautilus/7nrp/main/files/nrp-training.kubeconfig" -OutFile "$HOME\.kube\nrp-training.kubeconfig" ; `
$env:KUBECONFIG = "$HOME\.kube\config;$HOME\.kube\nrp-training.kubeconfig" ; `
kubectl config view --flatten | Out-File -Encoding ASCII "$HOME\.kube\config.merged" ; `
Move-Item -Force "$HOME\.kube\config.merged" "$HOME\.kube\config" ; `
$env:KUBECONFIG = "$HOME\.kube\config" ; `
kubectl config use-context nrp-training-k8s ; `
kubectl auth whoami
```
</details>
<div style="height: 2em;"></div>

After these three commands, `kubectl auth whoami` should print `system:serviceaccount:nrp-training:jupyterhub-sa`. Your default namespace is now `nrp-training-k8s` for every subsequent `kubectl` call — switch back to your prior context any time with `kubectl config use-context <name>`.

> **After the workshop ends:** this kubeconfig stops working. For ongoing NRP Nautilus access, configure `kubelogin` to use your personal **CILogon** identity instead of this temporary service account. Follow the official getting-started guide: <https://nrp.ai/documentation/userdocs/start/getting-started/>.

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

### kubectl flags you'll reach for constantly

| Flag | Purpose |
|---|---|
| `-n <namespace>` | Target a specific namespace. Drop it once your kubeconfig has a default namespace set. |
| `-l key=value` | Filter resources by label (e.g. `kubectl get pods -l app=hello-deploy-alice`). |
| `-A` / `--all-namespaces` | Show resources across every namespace you can read. |
| `-w` / `--watch` | Stream live updates instead of a one-shot list. Ctrl-C to stop. |
| `-o wide` | Add columns: node, pod IP, container image, etc. |
| `-o yaml` / `-o json` | Print the full resource manifest (great with `\| less`). |
| `-o jsonpath='{...}'` | Extract one field, e.g. `-o jsonpath='{.items[0].spec.nodeName}'`. |
| `--show-labels` | Append a column with every label a resource carries. |
| `--field-selector` | Filter on built-in fields (e.g. `--field-selector=status.phase=Running`). |
| `--previous` (on `kubectl logs`) | Logs from the *previous* container instance — essential for crashloops. |

### Gatekeeper: why every example sets requests and limits

Nautilus runs a cluster-wide Gatekeeper policy that **rejects pods that omit CPU or memory requests/limits, and rejects pods where the limit/request ratio exceeds 1.2×**. Every YAML in this tutorial sets `requests == limits` so you never trip it. If you copy-paste a manifest from upstream Kubernetes docs and it gets rejected, this is almost always why.

---

## GPUs on NRP

NRP carries many types of GPU. View live availability at <https://nrp.ai/viz/resources/>. Now that you have `kubectl`, you can also list every node and its GPU label directly:

```bash
kubectl get nodes -L nvidia.com/gpu.product
```

<details>
  <summary>Sample output</summary>

```
NAME                                         STATUS   ROLES            AGE      VERSION    GPU.PRODUCT
aarch64.calit2.optiputer.net                 Ready    <none>           2y234d   v1.33.8
admiralty-ncmir-mm-expanse-7d43bc97a0        Ready    cluster,master   13d
cenic-nrp1.hpc.cpp.edu                       Ready    <none>           17d      v1.33.8    NVIDIA-RTX-A6000
chi-dgx-node01.csuchico.edu                  Ready    <none>           165d     v1.33.8    Tesla-V100-SXM2-16GB
clu-fiona2.ucmerced.edu                      Ready    <none>           122d     v1.33.8    NVIDIA-GeForce-GTX-1080-Ti
…
```
</details>

The empty `GPU.PRODUCT` column is for CPU-only nodes; populated rows show what NVIDIA SKU each GPU node carries. We'll come back to this when we look at scheduling primitives later in the tutorial.

---

## Hands-on: kubectl basics and a simple pod

In this section we go through some common `kubectl` commands and create some Kubernetes objects.

YAML files are in this directory's [`yamls/`](yamls) folder. Find `test-pod.yaml` and open it in the editor.

### Creating a simple pod

Open `yamls/test-pod.yaml` and replace `<username>` (in `metadata.name`) with your NRP / GitHub handle. The full file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod-<username>
  namespace: nrp-training-k8s
spec:
  containers:
  - name: mypod
    image: ubuntu:22.04
    command: ["sh", "-c", "echo 'Hello from NRP!' && sleep 3600"]
    resources:
      limits:  { memory: 100Mi, cpu: 100m }
      requests: { memory: 100Mi, cpu: 100m }
```

Notice `requests` and `limits` are identical. Nautilus runs a Gatekeeper policy that rejects pods missing either, and rejects pods where the limit/request ratio exceeds 1.2. Setting them equal is the safe default.

Launch the pod:

```bash
kubectl apply -n nrp-training-k8s -f yamls/test-pod.yaml
```

<details>
<summary>Expected output</summary>

```
pod/test-pod-<username> created
```
</details>

Check whether the pod is running:

```bash
kubectl get pods -n nrp-training-k8s
kubectl get pod test-pod-<username> -n nrp-training-k8s -o wide
```

<details>
<summary>Expected output (after ~5–10 seconds)</summary>

```
NAME                  READY   STATUS    RESTARTS   AGE
test-pod-<username>   1/1     Running   0          12s
```
</details>

Look at its logs:

```bash
kubectl logs test-pod-<username> -n nrp-training-k8s
```

<details>
<summary>Expected output</summary>

```
Hello from NRP!
```
</details>

Run a command inside it (without an interactive shell):

```bash
kubectl exec test-pod-<username> -n nrp-training-k8s -- echo 'Command executed successfully'
```

Open an interactive shell (Ctrl-D to exit):

```bash
kubectl exec -it test-pod-<username> -n nrp-training-k8s -- /bin/bash
```

> **Debugging trio.** When something doesn't behave the way you expect, three commands are your defaults:
>
> ```bash
> kubectl describe pod test-pod-<username> -n nrp-training-k8s          # status + last events
> kubectl get events -n nrp-training-k8s --sort-by=.metadata.creationTimestamp | tail -20
> kubectl logs test-pod-<username> -n nrp-training-k8s --previous        # logs from the prior crash
> ```
>
> `describe` shows scheduling decisions, container state, and the last few events for that one resource. `get events` shows the full namespace timeline. `--previous` is essential for crashlooping pods — `logs` only shows the current container's stream.

Finally, clean up the pod to free resources:

```bash
kubectl delete pod test-pod-<username> -n nrp-training-k8s
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

## Hands-on: Persistent storage with a PVC

Pods are ephemeral — anything written to the container filesystem disappears when the pod terminates. **PersistentVolumeClaims** (PVCs) ask Kubernetes for a block of long-lived storage you can mount into pods. On NRP we typically use the `rook-ceph-block-east` storage class for general-purpose `ReadWriteOnce` block storage.

Open `yamls/pvc.yaml`. It contains both a 1 GiB PVC and a small writer pod that mounts it at `/data`. Replace `<username>` in both names, then apply:

```bash
kubectl apply -n nrp-training-k8s -f yamls/pvc.yaml
kubectl get pvc -n nrp-training-k8s
kubectl get pod pvc-pod-<username> -n nrp-training-k8s
```

<details>
<summary>Expected output (Ceph provisioning takes ~30–60s on first claim)</summary>

```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
pvc-<username>   Bound    pvc-99a63070-eb3d-490a-82fd-4e5811e4a5df   1Gi        RWO            rook-ceph-block-east   45s

NAME                 READY   STATUS    RESTARTS   AGE
pvc-pod-<username>   1/1     Running   0          47s
```
</details>

The PVC starts in `Pending` and flips to `Bound` once the volume is provisioned. Watch the writer pod append a line to the volume on every restart, then read it back:

```bash
kubectl logs -f pvc-pod-<username> -n nrp-training-k8s
# in another terminal:
kubectl exec pvc-pod-<username> -n nrp-training-k8s -- cat /data/log.txt
```

Now delete the pod and recreate just the pod (not the PVC) to prove the data survives. The simplest way is to delete only the pod and re-apply:

```bash
kubectl delete pod pvc-pod-<username> -n nrp-training-k8s
kubectl apply -n nrp-training-k8s -f yamls/pvc.yaml          # recreates the pod; the PVC is unchanged
kubectl exec pvc-pod-<username> -n nrp-training-k8s -- cat /data/log.txt   # the previous line is still there
```

**Don't delete the PVC yet** — the next section reuses `pvc-<username>` to demonstrate a multi-container pod. If you're skipping ahead, clean up with `kubectl delete -f yamls/pvc.yaml`.

> **Storage classes on NRP.** `rook-ceph-block-east` (RWO) is the default for single-pod block storage. For shared `ReadWriteMany` workloads use a CephFS class (`rook-cephfs`). See the [storage docs](https://nrp.ai/documentation/userdocs/storage/intro/) for the full list and when to use each.

---

## Hands-on: Multi-container pod (sidecar pattern, shared volume)

A pod can hold more than one container — they share the network namespace (same `localhost`) and any volumes you mount into both. This is the classic **sidecar** pattern: one container does the main job, a second container does something supporting (log shipping, file syncing, format conversion). Each container has its own logs, its own image, and its own resource requests.

`yamls/multicontainer.yaml` defines a pod with two containers that share the `pvc-<username>` claim from the previous section:
- **`writer`** appends a tick line to `/shared/data.log` every 5 seconds, and also echoes each tick to its own stdout.
- **`reader`** tails the same file from a different process and streams whatever shows up to **its** own stdout.

Before applying, delete the writer pod from the PVC section so the volume detaches — `rook-ceph-block-east` is `ReadWriteOnce`, so two pods on different nodes cannot mount it simultaneously:

```bash
kubectl delete pod pvc-pod-<username> -n nrp-training-k8s --ignore-not-found
```

Replace `<username>` in `multicontainer.yaml` (in the pod name **and** in `claimName`), then apply:

```bash
kubectl apply -n nrp-training-k8s -f yamls/multicontainer.yaml
kubectl get pod sidecar-<username> -n nrp-training-k8s
```

<details>
<summary>Expected output (READY shows 2/2 once both containers are up)</summary>

```
NAME                READY   STATUS    RESTARTS   AGE
sidecar-<username>  2/2     Running   0          25s
```
</details>

Inspect the container list:

```bash
kubectl get pod sidecar-<username> -n nrp-training-k8s -o jsonpath='{.spec.containers[*].name}' ; echo
```

<details>
<summary>Expected output</summary>

```
writer reader
```
</details>

Now read each container's log stream **separately** with `-c`:

```bash
kubectl logs sidecar-<username> -c writer -n nrp-training-k8s --tail=5
kubectl logs sidecar-<username> -c reader -n nrp-training-k8s --tail=5
# follow one of them in real time
kubectl logs -f sidecar-<username> -c reader -n nrp-training-k8s
```

<details>
<summary>Expected output</summary>

```
# writer (the producer — also echoed to its own stdout):
writer-tick 1 04:55:58
writer-tick 2 04:56:04
writer-tick 3 04:56:09

# reader (tailing the shared file from a different process):
reader started, tailing /shared/data.log
writer-tick 1 04:55:58
writer-tick 2 04:56:04
writer-tick 3 04:56:09
```
</details>

The reader's output proves both containers see the same file: every line the writer appends shows up in the reader's stream within a couple of seconds. Default `kubectl logs sidecar-<username>` (no `-c`) only works on single-container pods; on a multi-container pod kubectl will refuse and ask you to pick.

Clean up — this also releases the PVC if you no longer need it:

```bash
kubectl delete -n nrp-training-k8s -f yamls/multicontainer.yaml
kubectl delete -n nrp-training-k8s -f yamls/pvc.yaml
```

> **Why use a multi-container pod instead of two pods?** Two pods are isolated — no shared filesystem, no shared `localhost`. Use a single pod with two containers when the workloads are tightly coupled (a model server + its prometheus exporter, a training loop + a checkpoint shipper). Use separate pods otherwise.

---

## Hands-on: Configuration with ConfigMap, Secret, and env vars

Hard-coding paths, hostnames, or API tokens into images is a recipe for pain. Kubernetes gives you two purpose-built objects:
- **ConfigMap** — non-sensitive key/value config (hostnames, paths, feature flags). Stored as plain text in etcd.
- **Secret** — sensitive values (tokens, passwords, TLS keys). Stored base64-encoded in etcd, with separate RBAC; tooling treats them differently (kubectl won't print them by default, dashboards mask them).

You can mount either as files or expose them as environment variables. We'll do env vars here — the most common pattern.

`yamls/configmap-secret.yaml` ships three objects in one file: a ConfigMap (`GREETING`, `SERVER_PORT`), a Secret (`API_TOKEN`), and a Pod that pulls **all** ConfigMap keys in bulk via `envFrom` and pulls the Secret value via an explicit `secretKeyRef`. Replace `<username>` in all four names and apply:

```bash
kubectl apply -n nrp-training-k8s -f yamls/configmap-secret.yaml
kubectl get configmap app-config-<username> -n nrp-training-k8s
kubectl get secret    app-secret-<username> -n nrp-training-k8s
kubectl get pod       env-pod-<username>    -n nrp-training-k8s
```

Once the pod is running, check what it printed:

```bash
kubectl logs env-pod-<username>
```

<details>
<summary>Click to reveal expected output</summary>

```
GREETING=Hello from NRP
SERVER_PORT=8080
API_TOKEN starts with: tutorial…
```
</details>

You can confirm what's actually inside each object:

```bash
kubectl get configmap app-config-<username> -o yaml | grep -A2 '^data:'
kubectl get secret    app-secret-<username> -o jsonpath='{.data.API_TOKEN}' | base64 -d ; echo
```

Notice the Secret value comes back base64-encoded — that's storage format, not encryption. **Anyone who can `get secret` in your namespace can read it.** Treat ConfigMaps and Secrets as namespace-scoped: don't put production credentials in the workshop namespace.

Clean up:

```bash
kubectl delete -n nrp-training-k8s -f yamls/configmap-secret.yaml
```

> **Files vs env vars.** Mounting a Secret as a file (`volumeMounts` + `volumes.secret`) is preferable when the value is a multi-line credential (TLS cert, kubeconfig, JSON service account) or when you want updates to roll into the running pod automatically. Env vars are fixed at pod start.

---

## Hands-on: Deployment

A **Deployment** keeps a set of identical pods running. It spawns a ReplicaSet that creates the pods, restarts them when they fail, and rolls out new versions without downtime. Use it for long-running services (web apps, APIs, dashboards). Use a Job (next section) for one-shot work.

Open `yamls/deployment.yaml`, replace `<username>` everywhere, and apply:

```bash
kubectl apply -n nrp-training-k8s -f yamls/deployment.yaml
kubectl get deploy,rs,pod -n nrp-training-k8s -l app=hello-deploy-<username>
```

<details>
<summary>Expected output (after ~10s)</summary>

```
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-deploy-audit     2/2     2            2           14s

NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-deploy-audit-6f8c4bb7c7     2         2         2       14s

NAME                                            READY   STATUS    RESTARTS   AGE
pod/hello-deploy-audit-6f8c4bb7c7-9xw6m         1/1     Running   0          14s
pod/hello-deploy-audit-6f8c4bb7c7-ttgxv         1/1     Running   0          14s
```
</details>

You should see one Deployment, one ReplicaSet, and two Pods. Try the basic operations:

```bash
# scale to 4 replicas
kubectl scale deployment hello-deploy-<username> -n nrp-training-k8s --replicas=4
kubectl get pods -n nrp-training-k8s -l app=hello-deploy-<username>

# delete one specific pod and watch the Deployment immediately recreate it
VICTIM=$(kubectl get pod -n nrp-training-k8s -l app=hello-deploy-<username> -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod "$VICTIM" -n nrp-training-k8s
kubectl get pods -n nrp-training-k8s -l app=hello-deploy-<username>   # there are still 4 — a new pod replaced $VICTIM

# rolling update to a different image (nginx:alpine, swapping out nginxdemos/hello)
kubectl set image deployment/hello-deploy-<username> -n nrp-training-k8s hello=nginx:alpine
kubectl rollout status deployment/hello-deploy-<username> -n nrp-training-k8s
```

<details>
<summary>Expected rollout output</summary>

```
Waiting for deployment "hello-deploy-<username>" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "hello-deploy-<username>" rollout to finish: 1 old replicas are pending termination...
deployment "hello-deploy-<username>" successfully rolled out
```
</details>

**Don't delete this Deployment yet** — the next section uses one of its running pods to demonstrate `kubectl cp`, `port-forward`, and `patch`.

---

## Hands-on: Working with running pods (cp, port-forward, patch)

Three commands you'll reach for constantly once you have something running:

- **`kubectl cp`** — copy files into or out of a running container. Useful for shipping a dataset in, pulling logs/checkpoints out, or grabbing a coredump.
- **`kubectl port-forward`** — open a tunnel from your laptop's localhost to a port on a pod or Service. The fastest way to poke at an in-cluster web app without setting up an Ingress.
- **`kubectl patch`** — change one field on a live resource without re-applying its whole YAML. Useful for quick tweaks (replica count, image tag, label), debugging, or scripting.

We'll use a pod from the Deployment you applied above. Pick one:

```bash
POD=$(kubectl get pod -n nrp-training-k8s -l app=hello-deploy-<username> -o jsonpath='{.items[0].metadata.name}')
echo "$POD"
```

### kubectl cp

Copy a file **into** the pod, then verify with `exec`:

```bash
echo "training data v1" > /tmp/dataset.txt
kubectl cp /tmp/dataset.txt nrp-training-k8s/"$POD":/tmp/dataset.txt
kubectl exec "$POD" -n nrp-training-k8s -- cat /tmp/dataset.txt
```

<details>
<summary>Expected output</summary>

```
training data v1
```
</details>

Copy a file **out** of the pod:

```bash
kubectl exec "$POD" -n nrp-training-k8s -- sh -c 'echo "result $(date -u)" > /tmp/result.txt'
kubectl cp nrp-training-k8s/"$POD":/tmp/result.txt /tmp/result.txt
cat /tmp/result.txt
```

<details>
<summary>Expected output</summary>

```
tar: removing leading '/' from member names    ← warning, harmless
result Mon May  4 09:37:52 UTC 2026
```
</details>

> **Caveat:** `kubectl cp` requires `tar` to exist inside the container; minimal images (`alpine`, `distroless`) often don't have it. If you hit "tar not found", fall back to streaming via `kubectl exec` (e.g., `kubectl exec $POD -n nrp-training-k8s -- cat /path/to/file > local`).

### kubectl port-forward

Open a tunnel from `localhost:8080` on your laptop to port 80 on the pod:

```bash
kubectl port-forward "$POD" -n nrp-training-k8s 8080:80
# in another terminal:
curl -s http://localhost:8080 | head -3
# Ctrl-C in the first terminal to close the tunnel
```

<details>
<summary>Expected output (in the curl terminal)</summary>

```
Server address: 10.244.x.x:80
Server name: hello-deploy-<username>-6f8c4bb7c7-9xw6m
Date: 04/May/2026:09:37:52 +0000
```
</details>

Or forward against the **Deployment** instead of a specific pod (kubectl picks any ready pod from the Deployment for you, so the tunnel survives pod restarts):

```bash
kubectl port-forward deployment/hello-deploy-<username> -n nrp-training-k8s 8080:80
```

`port-forward` works on any pod with an open TCP port — it does **not** require a Service or an Ingress, doesn't go through HAProxy, and is per-user (no public URL). Perfect for "I just want to inspect this dashboard from my laptop."

### kubectl patch

Change one field without rewriting the whole YAML. Two patch styles:

```bash
# strategic merge patch (default) — bump replicas to 3
kubectl patch deployment hello-deploy-<username> -n nrp-training-k8s \
  -p '{"spec":{"replicas":3}}'
kubectl get deployment hello-deploy-<username> -n nrp-training-k8s

# add a label to the running deployment
kubectl patch deployment hello-deploy-<username> -n nrp-training-k8s \
  -p '{"metadata":{"labels":{"owner":"<username>","env":"workshop"}}}'

# JSON patch — most precise, used when you need to remove or replace at a specific path
kubectl patch deployment hello-deploy-<username> -n nrp-training-k8s --type=json \
  -p='[{"op":"replace","path":"/spec/replicas","value":2}]'
```

<details>
<summary>Expected output (each patch prints one confirmation line)</summary>

```
deployment.apps/hello-deploy-<username> patched
deployment.apps/hello-deploy-<username> patched
deployment.apps/hello-deploy-<username> patched
```
</details>

`kubectl edit deployment hello-deploy-<username> -n nrp-training-k8s` is the interactive cousin — it opens the current spec in `$EDITOR` and applies the diff when you save.

Now clean up the deployment:

```bash
kubectl delete -n nrp-training-k8s -f yamls/deployment.yaml
```

---

## Hands-on: Batch Job

A **Job** runs pods until a target number of them complete successfully, then stops. Use it for one-shot batch work: data processing, training runs, simulations. Failed pods are retried up to `backoffLimit`; successful pods are kept around (and pruned automatically `ttlSecondsAfterFinished` later).

Open `yamls/job.yaml`, replace `<username>`, and apply:

```bash
kubectl apply -n nrp-training-k8s -f yamls/job.yaml
kubectl get jobs -n nrp-training-k8s
kubectl get pods -n nrp-training-k8s -l job-name=pi-<username>
```

Stream the result (after about a minute of CPU work) and check completion:

```bash
kubectl logs -n nrp-training-k8s -l job-name=pi-<username>
kubectl get job pi-<username> -n nrp-training-k8s
```

<details>
<summary>Click to reveal expected output</summary>

```
3.14159265358979323846264338327950288419716939937510582097494459230781640628620…
```
```
NAME             STATUS     COMPLETIONS   DURATION   AGE
pi-<username>    Complete   1/1           53s        57s
```
(Duration depends on which CPU node the Job lands on — expect 50–120s.)
</details>

Note: `bpi(2000)` is CPU-bound — expect 50–120 seconds depending on which CPU node the Job lands on. The Job is auto-deleted 10 minutes after completion (`ttlSecondsAfterFinished: 600`). To clean up immediately:

```bash
kubectl delete -n nrp-training-k8s -f yamls/job.yaml
```

---

## Hands-on: Exposing a service over HTTPS (Deployment + Service + Ingress)

Pods aren't reachable from outside the cluster on their own. To expose an HTTP application you need three objects:
1. A **Deployment** that runs the pods.
2. A **Service** that gives those pods a stable in-cluster name and port.
3. An **Ingress** on the `haproxy` class that routes a public hostname (and its TLS cert) to the Service.

NRP runs HAProxy as the cluster ingress controller and **Cert Manager** with Let's Encrypt issues a free TLS certificate automatically for any `*.nrp-nautilus.io` hostname you pick. Full details: <https://nrp.ai/documentation/userdocs/running/ingress/>.

Open `yamls/ingress-demo.yaml` and replace **every** `<username>` (the hostname `hello-<username>.nrp-nautilus.io` must be globally unique). Apply the bundle:

```bash
kubectl apply -n nrp-training-k8s -f yamls/ingress-demo.yaml
kubectl get deploy,svc,ingress -n nrp-training-k8s -l k8s-app=hello-web-<username>
```

<details>
<summary>Expected output (after ~10–20 seconds)</summary>

```
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-web-<username>   2/2     2            2           18s

NAME                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/hello-web-<username>   ClusterIP   10.107.50.135   <none>        8080/TCP   18s

NAME                                            CLASS     HOSTS                              ADDRESS   PORTS     AGE
ingress.networking.k8s.io/hello-web-<username>   haproxy   hello-<username>.nrp-nautilus.io             80, 443   18s
```
</details>

Wait ~60 seconds for HAProxy to pick up the new Ingress and for Cert Manager + Let's Encrypt to issue the cert, then visit it from your laptop:

```bash
curl -sI https://hello-<username>.nrp-nautilus.io | head -5
```

<details>
<summary>Expected output</summary>

```
HTTP/2 200
server: nginx/1.29.1
date: Mon, 04 May 2026 09:42:43 GMT
content-type: text/plain
content-length: 165
```
</details>

Open the URL in your browser to see the demo page; reload a few times — the `Server name` line changes as the load balancer cycles between the two replicas.

If you only want to test the in-cluster Service without exposing it publicly, you can skip the Ingress and use a port-forward:

```bash
kubectl port-forward -n nrp-training-k8s svc/hello-web-<username> 8080:8080
# in another terminal:
curl http://localhost:8080
```

Clean up — this also releases the public hostname:

```bash
kubectl delete -n nrp-training-k8s -f yamls/ingress-demo.yaml
```

> **Using your own domain.** Point a CNAME at `nrp-nautilus.io` (or `east.nrp-nautilus.io`) and add a `tls.secretName` to the Ingress with a TLS Secret you provide. The [Ingress docs](https://nrp.ai/documentation/userdocs/running/ingress/) walk through both options, including auto-issuing your own cert via Cert Manager.

---

## Scheduling: node labels, nodeSelector / nodeAffinity, taints, and tolerations

NRP is a heterogeneous, shared cluster: 500+ nodes, hundreds of GPU SKUs, contributing institutions all over the country, and pools of nodes reserved for specific projects. **Scheduling primitives** are how you tell Kubernetes "*put my pod **here**, not **there**.*" The two halves of the contract are:

| Primitive | Lives on | Asks the question |
|---|---|---|
| **Node label** | Node | "What is this node? (GPU type, region, project owner, …)" |
| **`nodeSelector` / `nodeAffinity`** | Pod | "Which nodes am I willing to land on?" |
| **Taint** | Node | "Who is allowed to land here?" |
| **Toleration** | Pod | "I have permission to land on those tainted nodes." |

Labels + affinity are an **attraction** — pods *want* matching nodes. Taints + tolerations are a **repulsion** — tainted nodes reject pods that don't have a matching toleration. You usually need **both**: a toleration to be allowed onto a node, plus an affinity rule so the scheduler actually picks it. Toleration alone doesn't pull pods in; affinity alone doesn't get past a taint.

### The 7NRP reserved nodes

For the duration of the workshop, NRP has a pool of GPU nodes reserved for tutorial use. They are marked two ways:

- **Label** `nrp-training=true` — distinguishes them from every other node in the cluster.
- **Taint** `nautilus.io/reservation=nrp:NoSchedule` — keeps non-tutorial workloads off them.

`NoSchedule` is the strict effect: the scheduler refuses to place a pod on a tainted node unless the pod tolerates the taint. (The other effects are `PreferNoSchedule` — soft hint — and `NoExecute` — also evicts already-running pods.)

**Hands-on: explore the pool.**

```bash
# Which nodes are reserved for the workshop, and what GPU do they have?
kubectl get nodes -l nrp-training=true -L nvidia.com/gpu.product

# What taints are on them?
kubectl get nodes -l nrp-training=true \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'
```

<details>
<summary>Expected output</summary>

```
NAME                    STATUS   ROLES    AGE      VERSION    GPU.PRODUCT
gpu-03.nrp.mghpcc.org   Ready    <none>   3y195d   v1.33.10   NVIDIA-A10
gpu-11.nrp.mghpcc.org   Ready    <none>   3y195d   v1.33.8    NVIDIA-A10
gpu-12.nrp.mghpcc.org   Ready    <none>   3y192d   v1.33.8    NVIDIA-A10
…

gpu-03.nrp.mghpcc.org	[{"effect":"NoSchedule","key":"nautilus.io/reservation","value":"nrp"},{"effect":"PreferNoSchedule","key":"nvidia.com/gpu","value":"Exists"}]
gpu-11.nrp.mghpcc.org	[{"effect":"NoSchedule","key":"nautilus.io/reservation","value":"nrp"},{"effect":"PreferNoSchedule","key":"nvidia.com/gpu","value":"Exists"}]
…
```
</details>

You should see `nautilus.io/reservation=nrp:NoSchedule` on every reserved node.

### Targeting reserved nodes from a pod spec

To land your pod on the reserved pool, your YAML needs both a toleration (so the taint lets you in) **and** a nodeAffinity rule (so the scheduler picks one of those nodes specifically). The pattern — used throughout this repo — looks like this:

```yaml
spec:
  tolerations:
  - key: nautilus.io/reservation
    operator: Equal
    value: nrp
    effect: NoSchedule

  affinity:
    nodeAffinity:
      # SOFT: prefer reserved nodes; fall back to anywhere if none are free
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: nrp-training
            operator: In
            values: ["true"]
      # HARD alternative: fail to schedule unless on a reserved node
      # requiredDuringSchedulingIgnoredDuringExecution:
      #   nodeSelectorTerms:
      #   - matchExpressions:
      #     - key: nrp-training
      #       operator: In
      #       values: ["true"]
```

`required…` means "schedule **only** on matching nodes — pending forever otherwise." `preferred…` is a soft hint with a `weight` (1-100); the scheduler picks a matching node if one is free, but won't block your pod if everything is busy. **For workshop demos prefer `preferred`** — it keeps your pod from getting stuck if every reserved node is full.

`yamls/gpu-pod.yaml` already follows this pattern — open it and find the `tolerations:` + `affinity:` blocks. Tutorial 3's z2jh values use the same machinery (`extraTolerations` + `extraNodeAffinity`) so every JupyterHub user pod lands on the reserved pool too.

### Where this comes up beyond the workshop

- Targeting a specific GPU model (e.g. `nvidia.com/gpu.product=NVIDIA-A100-PCIE-40GB`) — same pattern, different label key.
- Targeting a region / institution (`topology.kubernetes.io/region=us-west`).
- Selecting a CUDA runtime version (`nvidia.com/cuda.runtime.major=12`).
- Avoiding spot/preemptible nodes (taint `node.kubernetes.io/spot=true:NoSchedule` — only pods that explicitly tolerate it land there).

The GPU pod section below shows the GPU-specific label keys.

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
kubectl apply -n nrp-training-k8s -f yamls/gpu-pod.yaml
# check that the pod is created
kubectl get pods -n nrp-training-k8s
```

Once the pod is in a ready state, exec into it:

```bash
kubectl exec -it tutorial-<username>-gpu-pod -n nrp-training-k8s -- /bin/bash
```

Try running `nvidia-smi` from within the pod.

**Important:** Terminate this pod when you are done — GPUs are scarce shared resources:

```bash
kubectl delete pod tutorial-<username>-gpu-pod -n nrp-training-k8s
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

## End — cleanup

**Please make sure you did not leave any running pods or services.** Completed Jobs are OK.

```bash
# delete anything you created in this part
kubectl delete pod test-pod-<username>            -n nrp-training-k8s --ignore-not-found
kubectl delete -f yamls/multicontainer.yaml       -n nrp-training-k8s --ignore-not-found
kubectl delete -f yamls/pvc.yaml                  -n nrp-training-k8s --ignore-not-found
kubectl delete -f yamls/configmap-secret.yaml     -n nrp-training-k8s --ignore-not-found
kubectl delete -f yamls/deployment.yaml           -n nrp-training-k8s --ignore-not-found
kubectl delete -f yamls/job.yaml                  -n nrp-training-k8s --ignore-not-found
kubectl delete -f yamls/ingress-demo.yaml         -n nrp-training-k8s --ignore-not-found
kubectl delete pod tutorial-<username>-gpu-pod    -n nrp-training-k8s --ignore-not-found

# what's left? (use this anytime — "what did I leave running?")
kubectl get all -n nrp-training-k8s
```

**Need help?** [Full docs](https://nrp.ai/documentation/) · [Matrix chat](https://nrp.ai/contact/) · [FAQ](https://nrp.ai/documentation/userdocs/start/faq/) · [Policies](https://nrp.ai/documentation/userdocs/start/policies/)

**Related docs:** [Using Nautilus](https://nrp.ai/documentation/userdocs/start/using-nautilus/) · [Kubernetes basics](https://nrp.ai/documentation/userdocs/tutorial/basic/) · [GPU pods](https://nrp.ai/documentation/userdocs/running/gpu-pods/) · [Run jobs](https://nrp.ai/documentation/userdocs/running/jobs/) · [Storage](https://nrp.ai/documentation/userdocs/storage/intro/) · [JupyterHub service](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/) · [Live resources](https://nrp.ai/viz/resources/)
