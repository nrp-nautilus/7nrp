# Setting Up Custom JupyterHubs for Classroom and Research

The primary access mechanism we support for classroom and research usage on NRP is via JupyterHub. This tutorial covers how instructors, TAs, and PIs can deploy and maintain custom JupyterHub instances. Topics include controlled access, custom software stacks/containers, and resource scaling — all driven by Helm.

YAMLs referenced in this tutorial live in this directory's [`yamls/`](yamls) folder.

> 📘 **Docs:** [Deploy JupyterHub](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub/) · [JupyterHub service](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/) · [Sci images](https://nrp.ai/documentation/userdocs/running/sci-img/) · [Build images](https://nrp.ai/documentation/userdocs/tutorial/images/) · [GitLab CI](https://nrp.ai/documentation/userdocs/development/gitlab/) · [Persistent storage](https://nrp.ai/documentation/userdocs/storage/ceph/) · [Z2JH (upstream)](https://z2jh.jupyter.org)

## Overview

This tutorial covers how to deploy and manage JupyterHub environments for groups, courses, or research labs on NRP.

## Key concepts

- **Helm Charts**: package managers for Kubernetes applications.
- **JupyterHub Helm Chart**: pre-configured deployment for JupyterHub.
- **Configuration**: customize JupyterHub through YAML configuration files.
- **Resource Management**: set CPU, memory, and GPU limits per user.
- **Storage**: configure persistent storage for user home directories.

---

# 1. Install Helm and kubectl

Helm is a package manager for Kubernetes — instead of authoring every Deployment, Service, and ConfigMap by hand, you install a chart (a reusable bundle of templates) and tune it through a values file. We'll use Helm to install JupyterHub from the official chart.

If you're working inside the [training JupyterHub](https://training.nrp-nautilus.io/), `kubectl`, `helm`, and the kubeconfig are already wired up in your terminal — open a terminal in JupyterLab, run `kubectl auth whoami && helm version --short` to confirm, then skip ahead to [section 2](#2-add-the-jupyterhub-helm-repository).

If you're running locally, install both tools and add the workshop kubeconfig with the three universal copy-paste commands below. None of them assume you have cloned this repo. The kubeconfig carries the `jupyterhub-sa` service-account token plus the cluster CA; the embedded token is valid for the duration of 7NRP, through end-of-day Thursday, May 7, 2026.

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

<details>
  <summary>Click to reveal expected output</summary>

```text
Username:   system:serviceaccount:nrp-training:jupyterhub-sa
v3.16.4+g7877b45
```
</details>

> **After the workshop ends:** this kubeconfig stops working. For ongoing NRP Nautilus access, configure `kubelogin` to use your personal **CILogon** identity instead of this temporary service account. Follow the official getting-started guide: <https://nrp.ai/documentation/userdocs/start/getting-started/>.

---

# 2. Add the JupyterHub Helm repository

- JupyterHub can only be deployed once per namespace.
- For the tutorial, we have pre-created namespaces for the registered participants so you can follow along.
- We have passed out slips of paper with a namespace name to each participant. Please use that namespace below where you see ```<namespace>```.
- Continue the execise using the terminal in the tutorial Jupyterhub server or from your computer's terminal.

**Please stick to using your pre-made namespace.**

```bash
# Add JupyterHub Helm repository
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/ -n <namespace>

# Update Helm repositories
helm repo update

# Verify the repository was added
helm repo list
```

<details>
<summary>Click to reveal output</summary>

```text
"jupyterhub" has been added to your repositories
```

```text
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jupyterhub" chart repository
Update Complete. ⎈Happy Helming!⎈
```

```text
NAME         URL
jupyterhub   https://jupyterhub.github.io/helm-chart/
```
</details>

---

# 3. Basic JupyterHub deployment

Full documentation: [https://nrp.ai/documentation/userdocs/jupyter/jupyterhub/](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub/).

## 3.1 Examine the values file

Open `yamls/jhub-values.yaml`. The full file is included in this part's `yamls/` directory; key sections:

```yaml
hub:
  config:
    JupyterHub:
      # Authentication and admin
      authenticator_class: dummy
      admin_access: true
      admin_users: ["admin"]
    DummyAuthenticator:
      password: "training123"
    Authenticator:
      allowed_users: set()
  service:
    type: ClusterIP
  db:
    type: sqlite-pvc
    pvc:
      accessModes: [ReadWriteOnce]
      storage: 1Gi
      storageClassName: rook-ceph-block-east
  resources:
    limits: {cpu: "2", memory: 1Gi}
    requests: {cpu: 100m, memory: 512Mi}
proxy:
  secretToken: 'secret_token'
  service:
    type: ClusterIP
# single user settings
singleuser:
  uid: 0
  fsGid: 100
  extraEnv: {GRANT_SUDO: "yes"}
  storage:
    type: dynamic
    capacity: 5Gi
    homeMountPath: /home/jovyan
    dynamic:
      storageClass: rook-ceph-block-east
      pvcNameTemplate: claim-{username}{servername}
      volumeNameTemplate: volume-{username}{servername}
      storageAccessModes: [ReadWriteOnce]
  image:
    name: quay.io/jupyter/scipy-notebook
    tag: 2024-04-22
  cpu: {limit: 3, guarantee: 3}
  memory: {limit: 10G, guarantee: 10G}
  defaultUrl: "/lab"
  profileList:
  - display_name: Scipy
    kubespawner_override:
      image_spec: quay.io/jupyter/scipy-notebook:2024-04-22
    default: True
  - display_name: Tensorflow
    kubespawner_override:
      image_spec: quay.io/jupyter/tensorflow-notebook:cuda-2024-04-22
  - display_name: Pytorch
    kubespawner_override:
      image_spec: quay.io/jupyter/pytorch-notebook:cuda12-2024-04-22
  # ...more profiles in yamls/jhub-values.yaml
# Required: automatically close inactive sessions 
cull:
  enabled: true
  timeout: 3600
  every: 600
```

## 3.2 Generate a secret token

Before deploying, generate a secret token for the proxy:

```bash
openssl rand -hex 32
```

**Important:** replace `secret_token` in `yamls/jhub-values.yaml` with the token generated above.

## 3.3 Deploy JupyterHub

Deploy via Helm. Replace `<namespace>` and `<release-name>` (e.g., `jhub-basic`):

```bash
helm upgrade --cleanup-on-fail --install <release-name> jupyterhub/jupyterhub \
  --namespace <namespace> \
  --version=3.3.7 \
  --values yamls/jhub-values.yaml \
  --wait \
  --timeout=10m
```

<details>
<summary>Click to reveal expected output</summary>

```text
Release "<release-name>" does not exist. Installing it now.
NAME: <release-name>
LAST DEPLOYED: Tue Mar 10 03:21:43 2026
NAMESPACE: <namespace>
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
       You have successfully installed the official JupyterHub Helm chart!

### Installation info

  - Kubernetes namespace: <namespace>
  - Helm release name:    <release-name>
  - Helm chart version:   3.3.7
  - JupyterHub version:   4.1.5
```
</details>

Now check what was deployed:

```bash
kubectl get pods -n <namespace>
kubectl get services -n <namespace>
kubectl get pvc -n <namespace>
```

You should see pods for the hub, proxy, and (when a user starts a server) user pods:
- **hub**: manages authentication, user sessions, and spawning user notebook servers.
- **proxy**: routes incoming traffic to the hub or to the correct user notebook server.
- **user pods**(once you launch a server): the individual Jupyter servers for each user.

You will also see persistent storage. A PVC for the hub is created at deployment, and PVCs for user servers are created whenever a new server is launched. If you have a shared storage that all pods should access, you can create a PVC and mount it via the Helm values.

<details>
<summary>Click to reveal expected pod/service/PVC output</summary>

```text
NAME                     READY   STATUS    RESTARTS   AGE
hub-5f5c7f8588-czp56     1/1     Running   0          26m
jupyter-admin            1/1     Running   0          13m
proxy-857f69dcbc-znhkd   1/1     Running   0          41m
```
```text
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hub            ClusterIP   10.110.96.59    <none>        8081/TCP   5m1s
proxy-api      ClusterIP   10.106.17.152   <none>        8001/TCP   5m1s
proxy-public   ClusterIP   10.110.183.21   <none>        80/TCP     5m1s
```
```text
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
claim-admin   Bound    pvc-9acc9f4d-2681-419a-963b-cb27ade541b6   5Gi        RWO            rook-ceph-block-east   14m
hub-db-dir    Bound    pvc-24ed6711-d5fc-4216-9689-f233b57818e5   1Gi        RWO            rook-ceph-block-east   42m
```
</details>

---

# 4. Configure Ingress

To make JupyterHub accessible via a URL, add an `ingress` section to `yamls/jhub-values.yaml` (or create a separate ingress file):

```yaml
ingress:
  enabled: true
  ingressClassName: haproxy
  hosts: ["<your-jupyterhub-name>.nrp-nautilus.io"]
  pathSuffix: ''
  tls:
    - hosts:
      - <your-jupyterhub-name>.nrp-nautilus.io
```

Update your deployment:

```bash
helm upgrade <release-name> jupyterhub/jupyterhub \
  --namespace <namespace> \
  --version=3.3.7 \
  --values yamls/jhub-values.yaml \
  --wait \
  --timeout=10m
```

Verify:

```bash
kubectl get pods -n <namespace>
kubectl get ingress -n <namespace>
```

<details>
<summary>Click to reveal expected output</summary>

```text
Release "<release-name>" has been upgraded. Happy Helming!
NAME: <release-name>
NAMESPACE: <namespace>
STATUS: deployed
REVISION: 2
NOTES:
  - Verify web based access:
    Try insecure HTTP access: http://<your-jupyterhub-name>.nrp-nautilus.io/
    Try secure HTTPS access:  https://<your-jupyterhub-name>.nrp-nautilus.io/
```

```text
NAME         CLASS     HOSTS                                       ADDRESS   PORTS     AGE
jupyterhub   haproxy   <your-jupyterhub-name>.nrp-nautilus.io                80, 443   2m30s
```
</details>

---

# 5. Advanced configuration

## 5.1 Multiple image profiles

Configure multiple image options for users to choose from:

```yaml
singleuser:
  profileList:
  - display_name: Scipy
    kubespawner_override:
      image_spec: quay.io/jupyter/scipy-notebook:2024-04-22
    default: True
  - display_name: R
    kubespawner_override:
      image_spec: quay.io/jupyter/r-notebook:2024-04-22
  - display_name: Julia
    kubespawner_override:
      image_spec: quay.io/jupyter/julia-notebook:2024-04-22
  - display_name: Tensorflow
    kubespawner_override:
      image_spec: quay.io/jupyter/tensorflow-notebook:cuda-2024-04-22
  - display_name: Pytorch
    kubespawner_override:
      image_spec: quay.io/jupyter/pytorch-notebook:cuda12-2024-04-22
  - display_name: Datascience (scipy, Julia, R)
    kubespawner_override:
      image_spec: quay.io/jupyter/datascience-notebook:2024-04-22
```

## 5.2 Resource limits per profile

Set different resource limits for different profiles:

```yaml
singleuser:
  profileList:
  - display_name: Small (2 CPU, 4GB RAM)
    kubespawner_override:
      image_spec: quay.io/jupyter/scipy-notebook:2024-04-22
      cpu_limit: 2
      cpu_guarantee: 2
      mem_limit: 4G
      mem_guarantee: 4G
  - display_name: Medium (4 CPU, 8GB RAM)
    kubespawner_override:
      image_spec: quay.io/jupyter/scipy-notebook:2024-04-22
      cpu_limit: 4
      cpu_guarantee: 4
      mem_limit: 8G
      mem_guarantee: 8G
  - display_name: Large (8 CPU, 16GB RAM)
    kubespawner_override:
      image_spec: quay.io/jupyter/scipy-notebook:2024-04-22
      cpu_limit: 8
      cpu_guarantee: 8
      mem_limit: 16G
      mem_guarantee: 16G
```

Try adding one or more of these profiles to your `jhub-values.yaml` and re-deploy. The change will be reflected in the spawner page.

## 5.3 Shared storage

If you have a shared storage volume that all users should access, add an `extraVolumes` section under `singleuser.storage`:

```yaml
singleuser:
  storage:
    type: dynamic
    extraLabels: {}
    extraVolumes:
      - name: jupyterhub-shared
        persistentVolumeClaim:
          claimName: jupyterhub-shared-volume
    extraVolumeMounts:
      - name: jupyterhub-shared
        mountPath: /home/shared
    capacity: 5Gi
    homeMountPath: /home/jovyan
    dynamic:
      storageClass: rook-ceph-block-east
      pvcNameTemplate: claim-{username}{servername}
      volumeNameTemplate: volume-{username}{servername}
      storageAccessModes: [ReadOnlyMany]
```

---

# 6. Managing JupyterHub

## 6.1 Check status

```bash
# List all Helm releases in your namespace
helm list -n <namespace>
```

<details>
<summary>Click to reveal output</summary>

```text
NAME              NAMESPACE  REVISION  UPDATED                              STATUS    CHART             APP VERSION
<release-name>    <ns>       2         2026-03-10 03:35:39.476432 -0400 EDT deployed  jupyterhub-3.3.7  4.1.5
```
</details>

```bash
# Check the hub pod logs
kubectl logs -n <namespace> -l app=jupyterhub,component=hub --tail=50
```

```bash
# Check all user pods
kubectl get pods -n <namespace> -l app=jupyterhub,component=singleuser-server
```

## 6.2 Uninstall

To remove JupyterHub (and optionally all user data):

```bash
# Uninstall JupyterHub (this deletes the hub and proxy, but NOT user data)
helm uninstall <release-name> -n <namespace>
```

```bash
# To also delete user PVCs (be careful — this deletes all user data!)
# kubectl delete pvc -n <namespace> -l app=jupyterhub,component=singleuser-storage
```

---

# 7. Troubleshooting

### JupyterHub not starting

```bash
# Check hub pod logs
kubectl logs -n <namespace> -l app=jupyterhub,component=hub

# Check proxy pod logs
kubectl logs -n <namespace> -l app=jupyterhub,component=proxy
```

### User pods not starting

```bash
# Check events
kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp

# Describe the failing pod
kubectl describe pod <pod-name> -n <namespace>
```

### Storage issues

```bash
# Check PVC status
kubectl get pvc -n <namespace>

# Check storage class
kubectl get storageclass
```

---

# 8. Advanced topic: Building images in GitLab

NRP provides GitLab integration for building container images and automating CI/CD pipelines. This section covers how to use GitLab for building and deploying container images on NRP.

For comprehensive documentation, see: [https://nrp.ai/documentation/userdocs/development/gitlab/](https://nrp.ai/documentation/userdocs/development/gitlab/).

## 8.1 Key features

- **Container image building**: build Docker images directly in GitLab CI/CD.
- **Kubernetes integration**: deploy applications to your namespace from GitLab pipelines.
- **Automated workflows**: set up CI/CD pipelines for your projects.
- **Private repositories**: support for private GitLab repositories.
- **Custom images**: build and use custom images in JupyterHub and other services.

## 8.2 Prerequisites

- Access to NRP GitLab instance.
- Namespace admin privileges (for deploying to Kubernetes).
- Understanding of Git and GitLab CI/CD basics.
- Docker / container image concepts.

## 8.3 Getting started with GitLab

### Access GitLab

NRP provides a GitLab instance for building images and managing repositories. Access it through:
- The NRP Portal.
- A direct link provided by your namespace administrator.

### Create a project

1. Create a new GitLab project or use an existing one.
2. Add your code and Dockerfile.
3. Configure a CI/CD pipeline using `.gitlab-ci.yml`.

## 8.4 Building container images

### Basic GitLab CI/CD pipeline

Create a `.gitlab-ci.yml` in your repository:

```yaml
image: ghcr.io/osscontainertools/kaniko:debug

stages:
- build-and-push

build-and-push-job:
  stage: build-and-push
  variables:
    GODEBUG: "http2client=0"
  script:
  - echo "{\"auths\":{\"$CI_REGISTRY\":{\"username\":\"$CI_REGISTRY_USER\",\"password\":\"$CI_REGISTRY_PASSWORD\"}}}" > /kaniko/.docker/config.json
  - /kaniko/executor --cache=true --push-retry=10 --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA --destination $CI_REGISTRY_IMAGE:latest
```

### Using built images

Once your image is built you can use it in:
- **JupyterHub**: configure custom images in your JupyterHub deployment.
- **Kubernetes pods**: reference the image in your pod specifications.
- **Jobs**: use in batch jobs and other workloads.

## 8.5 Kubernetes integration

GitLab can deploy directly to your Kubernetes namespace:
1. **Create a service account** with appropriate permissions.
2. **Configure GitLab** with Kubernetes cluster information.
3. **Add deployment stages** to your CI/CD pipeline.

For detailed steps, see the [GitLab Kubernetes Integration documentation](https://nrp.ai/documentation/userdocs/development/gitlab/).

## 8.6 Best practices

- **Use tags**: tag images with version numbers or commit SHAs.
- **Cache layers**: optimize Docker builds with layer caching.
- **Security**: keep your GitLab tokens and credentials secure.
- **Testing**: test images before deploying to production.
- **Documentation**: document your build process and image usage.

## 8.7 Resources

- [NRP GitLab documentation](https://nrp.ai/documentation/userdocs/development/gitlab/)
- [GitLab CI/CD documentation](https://docs.gitlab.com/ee/ci/)
- [Docker best practices](https://docs.docker.com/develop/dev-best-practices/)

---

# End — cleanup

When you are done with this part, uninstall any Helm releases you created so the cluster is left clean for the next user:

```bash
helm uninstall <release-name> -n <namespace>
```

User PVCs are kept by default. Only delete them if you are sure you no longer need user data:

```bash
kubectl delete pvc -n <namespace> -l app=jupyterhub,component=singleuser-storage
```

**Need help?** [Full docs](https://nrp.ai/documentation/) · [Matrix chat](https://nrp.ai/contact/) · [FAQ](https://nrp.ai/documentation/userdocs/start/faq/) · [Live resources](https://nrp.ai/viz/resources/)

**Related docs:** [Deploy JupyterHub](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub/) · [JupyterHub service](https://nrp.ai/documentation/userdocs/jupyter/jupyterhub-service/) · [Sci images](https://nrp.ai/documentation/userdocs/running/sci-img/) · [Container fundamentals](https://nrp.ai/documentation/userdocs/tutorial/docker/) · [Build images](https://nrp.ai/documentation/userdocs/tutorial/images/) · [NRP GitLab CI](https://nrp.ai/documentation/userdocs/development/gitlab/) · [Persistent storage](https://nrp.ai/documentation/userdocs/storage/ceph/) · [Z2JH (upstream)](https://z2jh.jupyter.org)
