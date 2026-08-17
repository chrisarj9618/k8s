# Argo CD + NGINX GitOps Demo

This project demonstrates a basic **GitOps deployment using Argo CD and Kubernetes**.

The goal is to understand how Argo CD continuously compares the desired state stored in Git with the actual state running inside Kubernetes.

## Prerequisites


* Windows System
* Docker Desktop
* Minikube
* kubernetes Extension enabled in Docker Desktop
* GitHUB
* Argo CD

For this exercise, Minikube is used as the Kubernetes cluster.

## Architecture

```text
                Git Repository
                     │
                     │ nginx.yaml
                     ▼
                  Argo CD
                     │
                     │ Sync / Reconcile
                     ▼
                  Minikube
                     │
                     ▼
               NGINX Deployment
                     │
                ┌────┴────┐
                │         │
              Pod       Pod
```






---

# 1. Repository Structure

The Git repository is called `k8s`.

The project structure is:

```text
k8s/
└── nginx-argocd/
    └── nginx.yaml
```

The Argo CD application points to:

```text
nginx-argocd
```

as the repository path.

> The repository name `k8s` is not included in the path because the repository itself is already treated as the root directory by Argo CD.

---

# 2. Start Minikube

Check the Minikube status:

```powershell
minikube status
```

If Minikube is stopped, start it:

```powershell
minikube start
```

Verify the Kubernetes node:

```powershell
kubectl get nodes
```

Expected result:

```text
NAME       STATUS   ROLES           AGE
minikube   Ready    control-plane   ...
```

Also verify all system pods:

```powershell
kubectl get pods -A
```

---

# 3. Install Argo CD

Create the Argo CD namespace:

```powershell
kubectl create namespace argocd
```

Install Argo CD:

```powershell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Check Argo CD pods:

```powershell
kubectl get pods -n argocd
```

Wait until the Argo CD components are running.

---

# 4. Get Argo CD Admin Password

For Windows PowerShell, retrieve the initial admin password using:

```powershell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

In VSCODE 

```VSCODE
 kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{ [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

For Macos and Visual Studio Code or linux terminal 
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
The output is the initial password for the `admin` user.

Username:

```text
admin
```

---

# 5. Access Argo CD UI

Run port forwarding:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open the following URL in your browser:

```text
https://localhost:8080
```

Login using:

```text
Username: admin
Password: <initial admin password>
```

A browser certificate warning may appear because this is a local development environment.

---

# 6. NGINX Kubernetes Manifest

The `nginx.yaml` file contains an NGINX Deployment and Service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
          image: nginx:latest
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

The initial desired state is:

```text
NGINX replicas = 2
```

---

# 7. Create Argo CD Application

From the Argo CD UI, select:

```text
+ NEW APP
```

Configure the application as follows.

## General

```text
Application Name: nginx
Project:          default
Sync Policy:      Manual
```

## Source

```text
Repository URL: <your Git repository URL>
Revision:       HEAD
Path:           nginx-argocd
```

## Destination

```text
Cluster:   https://kubernetes.default.svc
Namespace: default
```

The important part is the repository path:

```text
nginx-argocd
```

Argo CD will look for Kubernetes manifests inside:

```text
k8s/
└── nginx-argocd/
    └── nginx.yaml
```

---

# 8. First Deployment

After creating the Argo CD application, the application may initially show:

```text
Status: OutOfSync
Health: Missing
```

This is expected.

It means:

```text
Git:
NGINX Deployment + Service
        │
        ▼
     Argo CD
        │
        ▼
Kubernetes:
Resources do not exist yet
```

Click:

```text
SYNC
```

Then:

```text
SYNCHRONIZE
```

After successful synchronization:

```text
Status: Synced
Health: Healthy
```

Verify from PowerShell:

```powershell
kubectl get pods
```

Expected:

```text
NAME                     READY   STATUS
nginx-xxxxxxxxxx-xxxxx   1/1     Running
nginx-xxxxxxxxxx-xxxxx   1/1     Running
```

Verify the Service:

```powershell
kubectl get svc
```

---

# 9. Understanding GitOps

The fundamental GitOps model is:

```text
                 Git
                  │
                  │ Desired State
                  ▼
               Argo CD
                  │
                  │ Reconciliation
                  ▼
             Kubernetes
                  │
                  │ Actual State
                  ▼
                NGINX
```

Git contains the desired configuration.

Kubernetes contains the actual running configuration.

Argo CD continuously compares the two.

---

# 10. Test 1 — Change Replicas in Git

Change:

```yaml
replicas: 2
```

to:

```yaml
replicas: 3
```

Commit and push the change.

Argo CD will detect that the Git state differs from the Kubernetes state.

The application will show:

```text
OutOfSync
```

because:

```text
Git             Kubernetes

3 replicas      2 replicas
     │                │
     └── Difference ──┘
```

With Manual Sync enabled, Argo CD will not automatically apply the change.

Click:

```text
SYNC
```

After synchronization:

```text
Git             Kubernetes

3 replicas      3 replicas
```

Verify:

```powershell
kubectl get pods
```

You should now have 3 NGINX pods.

---

# 11. Test 2 — Create Kubernetes Drift

Now manually change Kubernetes without changing Git.

Run:

```powershell
kubectl scale deployment nginx --replicas=1
```

Check:

```powershell
kubectl get pods
```

Kubernetes now has:

```text
1 replica
```

But Git still says:

```text
3 replicas
```

Therefore Argo CD detects:

```text
OutOfSync
```

This is called **configuration drift**.

```text
Desired State             Actual State

Git                        Kubernetes
replicas: 3                replicas: 1
     │                           │
     └──────── Argo CD ──────────┘
                    │
                    ▼
                 OutOfSync
```

---

# 12. Enable Automated Sync

Argo CD can automatically synchronize changes from Git.

In the Argo CD application:

```text
Application
   ↓
Details
   ↓
Sync Policy
```

Enable:

```text
Automated Sync
```

Now Argo CD can automatically apply changes detected in Git.

---

# 13. Enable Self-Heal

Enable:

```text
Self Heal
```

The configuration becomes:

```text
Automated Sync: YES
Self Heal:      YES
Prune:          NO
```

Self-Heal allows Argo CD to correct changes made directly inside Kubernetes.

For example:

```text
Git
replicas: 3
     │
     ▼
Argo CD
     │
     ▼
Kubernetes
replicas: 1
     │
     │ Detect Drift
     ▼
Self-Heal
     │
     ▼
Kubernetes
replicas: 3
```

Verify:

```powershell
kubectl get pods
```

Kubernetes should eventually return to 3 NGINX pods.

---

# 14. Test Self-Healing

Manually scale the Deployment:

```powershell
kubectl scale deployment nginx --replicas=5
```

Watch the pods:

```powershell
kubectl get pods -w
```

Because Git still defines:

```yaml
replicas: 3
```

Argo CD detects the drift and, with Self-Heal enabled, reconciles Kubernetes back to:

```text
3 replicas
```

---

# 15. Important Argo CD Concepts

## Desired State

The configuration stored in Git.

Example:

```yaml
replicas: 3
```

## Actual State

What is currently running inside Kubernetes.

Example:

```text
3 Pods
```

## Sync

The process of applying the desired Git state to Kubernetes.

## OutOfSync

The Git state and Kubernetes state are different.

## Synced

The Git state and Kubernetes state match.

## Health

Indicates whether the Kubernetes resources are functioning correctly.

## Automated Sync

Automatically applies changes from Git.

## Self-Heal

Corrects manual changes made directly to Kubernetes.

## Prune

Removes Kubernetes resources that no longer exist in Git.

---

# 16. Manual Sync vs Automated Sync

### Manual Sync

```text
Git Change
    │
    ▼
Argo CD detects change
    │
    ▼
OutOfSync
    │
    ▼
Human clicks SYNC
    │
    ▼
Kubernetes updated
```

### Automated Sync

```text
Git Change
    │
    ▼
Argo CD detects change
    │
    ▼
Automatic Sync
    │
    ▼
Kubernetes updated
```

### Automated Sync + Self-Heal

```text
Git Change
    │
    ▼
Automatic Sync
    │
    ▼
Kubernetes

        OR

Manual Kubernetes Change
    │
    ▼
Drift detected
    │
    ▼
Self-Heal
    │
    ▼
Desired Git State restored
```

---

# 17. Useful Commands

Check Minikube:

```powershell
minikube status
```

Check Kubernetes nodes:

```powershell
kubectl get nodes
```

Check all pods:

```powershell
kubectl get pods -A
```

Check NGINX pods:

```powershell
kubectl get pods
```

Check NGINX Deployment:

```powershell
kubectl get deployment nginx
```

Check NGINX Service:

```powershell
kubectl get svc nginx
```

Watch pods:

```powershell
kubectl get pods -w
```

Check Argo CD resources:

```powershell
kubectl get pods -n argocd
```

Port-forward Argo CD:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

# 18. Current Learning Architecture

At this stage, the environment is:

```text
┌──────────────────────────────┐
│          GitLab              │
│                              │
│  k8s/                        │
│   └── nginx-argocd/          │
│        └── nginx.yaml        │
└──────────────┬───────────────┘
               │
               │ Git
               ▼
┌──────────────────────────────┐
│          Argo CD             │
│                              │
│        Application           │
│           nginx              │
└──────────────┬───────────────┘
               │
               │ Sync
               ▼
┌──────────────────────────────┐
│          Minikube            │
│                              │
│      NGINX Deployment        │
│             │                │
│       ┌─────┴─────┐          │
│       ▼           ▼          │
│    NGINX Pod   NGINX Pod     │
│                              │
└──────────────────────────────┘
```

---

# 19. Next Learning Steps

After understanding this basic deployment, the recommended progression is:

1. **Prune**

   * Delete a Kubernetes manifest from Git
   * Understand what happens to the resource

2. **Argo CD CLI**

   * Login
   * List applications
   * Sync applications
   * View application status

3. **Helm + Argo CD**

   * Deploy NGINX using a Helm chart
   * Understand `values.yaml`

4. **GitLab CI/CD + Argo CD**

   * Build application
   * Push image to Amazon ECR
   * Update GitOps repository
   * Let Argo CD deploy

5. **Environment Management**

   * Dev
   * SIT
   * UAT
   * Production

6. **Argo CD Projects**

   * RBAC
   * Repository restrictions
   * Cluster restrictions
   * Namespace restrictions

7. **Secrets Management**

   * Kubernetes Secrets
   * AWS Secrets Manager
   * External Secrets Operator

8. **Production EKS Architecture**

   * Argo CD on EKS
   * Multiple applications
   * Multiple environments
   * ApplicationSets
   * Multi-cluster deployments

---

## Key Takeaway

The most important concept from this exercise is:

```text
              Git
               │
               │ Desired State
               ▼
            Argo CD
               │
               │ Reconcile
               ▼
          Kubernetes
               │
               │ Actual State
               ▼
          Running App
```

**Git is the source of truth.**

**Argo CD continuously works to make Kubernetes match that source of truth.**
