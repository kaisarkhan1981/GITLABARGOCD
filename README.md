# End-to-End GitOps Setup — Native GitLab on Ubuntu VM, GitLab Runner, Argo CD on Kubernetes, Nginx Application, and Webhook Integration

**Author:** Md Kaisar Ahmed Khan
**Date:** 19/10/2026
**Version:** 1.0

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Install GitLab on Ubuntu VM (Native)](#3-install-gitlab-on-ubuntu-vm-native)
4. [Install and Register GitLab Runner (Native)](#4-install-and-register-gitlab-runner-native)
5. [Install Argo CD in Kubernetes](#5-install-argo-cd-in-kubernetes)
6. [Create Nginx Manifests in GitLab](#6-create-nginx-manifests-in-gitlab)
7. [Create Argo CD Application via GUI](#7-create-argo-cd-application-via-gui)
8. [Configure Webhook (GitLab → Argo CD)](#8-configure-webhook-gitlab--argo-cd)
9. [Verification Checklist](#9-verification-checklist)
10. [Summary](#10-summary)

---

## 1. Architecture Overview

This document describes a complete GitOps pipeline with the following components:

| Component | Location | Purpose |
|-----------|----------|---------|
| GitLab (Omnibus) | Ubuntu VM (native install) | Source control, hosts manifests |
| GitLab Runner | Same Ubuntu VM (native install) | Executes CI/CD jobs |
| Argo CD | Kubernetes cluster | Continuous deployment (GitOps) |
| Nginx App | Kubernetes | Sample application deployed via Argo CD |
| Webhook | GitLab → Argo CD | Instant sync on push |

**Flow:** Developer pushes to GitLab → GitLab fires webhook → Argo CD refreshes → Kubernetes reconciles → Nginx deployed/updated.

---

## 2. Prerequisites

- Ubuntu 20.04 / 22.04 / 24.04 VM with at least **4 GB RAM, 2 vCPU, 20 GB disk**.
- A registered domain name (or static IP) pointing to the VM.
- A working Kubernetes cluster with `kubectl` configured.
- Root/sudo access on the VM.
- Network reachability between the GitLab VM and the Argo CD server (for webhooks).

---

## 3. Install GitLab on Ubuntu VM (Native)

### Step 1.1 — Update system and install dependencies

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl openssh-server ca-certificates tzdata perl
```

*Explanation:* Ensures the OS is current and installs required libraries for GitLab Omnibus.

### Step 1.2 — Configure firewall (if UFW is active)

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

*Explanation:* SSH (22) for git-over-ssh; HTTP/HTTPS (80/443) for the web UI and API.

### Step 1.3 — Add the official GitLab repository

```bash
curl --location "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh" | sudo bash
```

*Explanation:* Adds GitLab CE package source to apt. Use `gitlab-ee` for the Enterprise Edition.

### Step 1.4 — Install GitLab

```bash
sudo EXTERNAL_URL="http://<YOUR_VM_IP_OR_DOMAIN>" apt install gitlab-ce
```

*Explanation:* `EXTERNAL_URL` sets the GitLab instance URL used in clones, webhooks, and the UI. Replace `<YOUR_VM_IP_OR_DOMAIN>` with e.g. `http://192.168.1.50` or `https://gitlab.example.com`.

### Step 1.5 — Retrieve the initial root password

```bash
sudo cat /etc/gitlab/initial_root_password
```

*Explanation:* Log in as `root` with this password. The file auto-deletes after 24 hours.

---

## 4. Install and Register GitLab Runner (Native)

### Step 2.1 — Add runner repository and install

```bash
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install -y gitlab-runner
```

*Explanation:* Installs the runner agent natively on the Ubuntu VM.

### Step 2.2 — Register the runner

Get the registration token from **GitLab UI → Admin Area → CI/CD → Runners**.

```bash
sudo gitlab-runner register
```

Answer the prompts:

- **GitLab URL:** `http://<YOUR_VM_IP>`
- **Registration token:** *(paste token)*
- **Description:** `ubuntu-native-runner`
- **Tags:** `shell`
- **Executor:** `shell`

*Explanation:* The `shell` executor runs jobs directly on the Ubuntu host. Switch to `docker` if you prefer isolated build containers (Docker must be installed).

---

## 5. Install Argo CD in Kubernetes

### Step 3.1 — Install manifests

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

*Explanation:* Deploys all Argo CD CRDs, controllers, and services into the `argocd` namespace.

### Step 3.2 — Access the UI and get the admin password

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

*Explanation:* Login at `https://localhost:8080` with username `admin` and the retrieved password.

> **Important:** For webhooks to work, Argo CD must be reachable from the GitLab VM. Expose it via **NodePort, LoadBalancer, or Ingress** — not just localhost.

---

## 6. Create Nginx Manifests in GitLab

### Step 4.1 — Create a project in GitLab

- Log into GitLab as `root`.
- **New Project → Create blank project** → name it `nginx-gitops`.

### Step 4.2 — Add manifest files

**`manifests/nginx-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

**`manifests/nginx-service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

*Explanation:* A minimal Deployment (2 replicas) and ClusterIP Service. Change `type` to `NodePort`/`LoadBalancer` for external access.

### Step 4.3 — Push to GitLab

```bash
git clone http://<YOUR_VM_IP>/root/nginx-gitops.git
cd nginx-gitops
# add the manifests/ directory and files
git add .
git commit -m "Add nginx manifests"
git push origin main
```

---

## 7. Create Argo CD Application via GUI

1. Open the Argo CD UI.
2. Click **+ NEW APP**.
3. **General:**
   - Name: `nginx-demo`
   - Project: `default`
4. **Source:**
   - Repository URL: `http://<YOUR_VM_IP>/root/nginx-gitops.git`
   - Revision: `HEAD`
   - Path: `manifests`
5. **Destination:**
   - Cluster URL: `https://kubernetes.default.svc`
   - Namespace: `default`
6. **Sync Policy:** `Automatic` — check **Prune Resources** and **Self Heal**.
7. Click **Create**.

*Explanation:* Argo CD pulls the manifests from GitLab and reconciles them into the cluster. The UI shows **Synced/OutOfSync** status and health.

---

## 8. Configure Webhook (GitLab → Argo CD)

### Step 6.1 — (Recommended) Set the webhook secret in Argo CD

```bash
kubectl edit secret argocd-secret -n argocd
```

Add under `stringData`:

```yaml
webhook.gitlab.secret: my-super-secret-token
```

### Step 6.2 — Configure the webhook in GitLab

1. Project → **Settings → Webhooks**.
2. **URL:** `http://<ARGOCD_REACHABLE_IP>:<PORT>/api/webhook`
3. **Secret token:** *(same value as Step 6.1)*
4. **Trigger:** ✅ **Push events**
5. Click **Add webhook**.

*Explanation:* On every push, GitLab POSTs to Argo CD's `/api/webhook`. Argo CD validates the secret and refreshes the application immediately (bypassing the default 3-minute poll).

### Step 6.3 — Verify

```bash
# From the GitLab VM, verify reachability
curl -I http://<ARGOCD_REACHABLE_IP>:<PORT>/api/webhook
```

Then push a change (e.g., `replicas: 3`) and watch the Argo CD UI update within seconds.

**Troubleshooting:**

- **Red exclamation in GitLab:** The Argo CD URL isn't reachable from the VM. Test with `curl`.
- **Check Argo CD side:**

  ```bash
  kubectl logs -n argocd deployment/argocd-server | grep webhook
  ```

---

## 9. Verification Checklist

| # | Item | Verification |
|---|------|--------------|
| 1 | GitLab UI reachable | `http://<VM_IP>` loads |
| 2 | Runner online | Admin → Runners → green dot |
| 3 | Argo CD installed | `kubectl get pods -n argocd` all Running |
| 4 | Manifests in GitLab | Repo shows `manifests/*.yaml` |
| 5 | Argo CD app created | UI shows **Synced / Healthy** |
| 6 | Nginx running | `kubectl get pods -l app=nginx` |
| 7 | Webhook fires | Push change → instant refresh in UI |

---

## 10. Summary

You have configured:

- A **native GitLab CE** instance on Ubuntu hosting source code.
- A **native GitLab Runner** executing CI jobs.
- **Argo CD** in Kubernetes monitoring the GitLab repository.
- An **Nginx** application deployed via GitOps.
- A **webhook** from GitLab to Argo CD for instant sync.

This pipeline enables any Git commit to automatically propagate to the Kubernetes cluster within seconds of the push.

---

make .md file
