# 🚀 Jenkins on Kubernetes (kubeadm) CI/CD: GitHub → AWS ECR → Kubernetes Deployment

> **End-to-end DevOps CI/CD project** using Jenkins running on a kubeadm Kubernetes cluster.
> Automated build + push to **AWS ECR** and auto-deploy to Kubernetes on every GitHub push via Webhook.

---

## 📌 Table of Contents

* [Overview](#-overview)
* [Architecture](#-architecture)
* [Tech Stack](#-tech-stack)
* [Prerequisites](#-prerequisites)
* [Repository Structure](#-repository-structure)
* [Implementation Steps](#-implementation-steps)

  * [1) Kubernetes Pre-check](#1-kubernetes-pre-check)
  * [2) Create Jenkins Namespace](#2-create-jenkins-namespace)
  * [3) StorageClass Setup using NFS Dynamic Provisioner](#3-storageclass-setup-using-nfs-dynamic-provisioner)
  * [4) Install Jenkins using Helm](#4-install-jenkins-using-helm)
  * [5) RBAC for Jenkins Kubernetes Deployments](#5-rbac-for-jenkins-kubernetes-deployments)
  * [6) AWS ECR Repository Setup](#6-aws-ecr-repository-setup)
  * [7) Configure AWS Credentials in Jenkins](#7-configure-aws-credentials-in-jenkins)
  * [8) Jenkins Pipeline from GitHub (SCM Integration)](#8-jenkins-pipeline-from-github-scm-integration)
  * [9) GitHub Webhook Integration](#9-github-webhook-integration)
  * [10) Validate Deployment](#10-validate-deployment)
* [Screenshots](#-screenshots)
* [Troubleshooting](#-troubleshooting)
* [Future Enhancements](#-future-enhancements)
* [Resume Bullet Points](#-resume-bullet-points)

---

## ✅ Overview

This project implements a complete CI/CD pipeline:

* ✅ Jenkins deployed **inside Kubernetes** using Helm
* ✅ Jenkins uses **Persistent Volume** via NFS dynamic provisioning
* ✅ Jenkins pipeline executes using **Kubernetes Agent Pods**
* ✅ On every GitHub push, Jenkins pipeline triggers automatically via **Webhook**
* ✅ Pipeline builds Docker image and pushes to **AWS ECR**
* ✅ Pipeline deploys/updates application on Kubernetes automatically

---

## 🏗 Architecture

```text
Developer Push → GitHub Repo
        ↓ (Webhook)
Jenkins Controller (Kubernetes)
        ↓
Jenkins Agent Pod (Kubernetes)
        ↓
Docker Build
        ↓
Push Image → AWS ECR
        ↓
Deploy/Update → Kubernetes namespace
        ↓
Expose App → Service / Ingress (ELB)
```

---

## 🧰 Tech Stack

* **Kubernetes** (kubeadm)
* **Helm**
* **Jenkins** (Helm chart)
* **NFS Dynamic Provisioner** (`nfs-subdir-external-provisioner`)
* **AWS ECR**
* **GitHub Webhooks**
* **Nginx Ingress Controller**

---

## ✅ Prerequisites

### Kubernetes

* kubeadm Kubernetes cluster ready
* `kubectl` configured and working

Verify:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

### Tools Needed

* Helm 3.x
* AWS CLI configured (for ECR repo creation)
* Git

---

## 📂 Repository Structure

GitHub Repo:

```text
https://github.com/BhushanKhutle/jenkins-ecr-k8s-demo.git
```

Structure:

```text
jenkins-ecr-k8s-demo/
├── Dockerfile
├── index.html
└── Jenkinsfile
```

---

## 🛠 Implementation Steps

### 1) Kubernetes Pre-check

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get sc
```

---

### 2) Create Jenkins Namespace

```bash
kubectl create ns jenkins
```

---

### 3) StorageClass Setup using NFS Dynamic Provisioner

> Cluster had **no StorageClass**, so PVCs were Pending. We created dynamic provisioning using NFS.

#### 3.1 Setup NFS Server (RHEL node)

```bash
yum install -y nfs-utils rpcbind
systemctl enable --now rpcbind
systemctl enable --now nfs-server

mkdir -p /mnt/k8s-nfs
chmod -R 777 /mnt/k8s-nfs

cat <<EOF > /etc/exports
/mnt/k8s-nfs *(rw,sync,no_root_squash,no_subtree_check)
EOF

exportfs -rav
showmount -e localhost
```

#### 3.2 Install NFS Provisioner

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  -n kube-system \
  --set nfs.server=<NFS_SERVER_IP> \
  --set nfs.path=/mnt/k8s-nfs
```

Verify StorageClass:

```bash
kubectl get sc
kubectl get pods -n kube-system | grep nfs
```

---

### 4) Install Jenkins using Helm

#### 4.1 Add Jenkins Helm repo

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

#### 4.2 Create `values.yaml`

> Note: Helm chart changed older keys:
>
> * ❌ `controller.adminUser`
> * ✅ `controller.admin.username`

```yaml
controller:
  admin:
    username: admin
    password: "Jenkins@123"

  serviceType: NodePort
  nodePort: 32000

persistence:
  enabled: true
  size: 10Gi
  storageClass: "<YOUR_STORAGECLASS_NAME>"
```

#### 4.3 Install Jenkins

```bash
helm install jenkins jenkins/jenkins -n jenkins -f values.yaml
```

Verify:

```bash
kubectl get pods -n jenkins -w
kubectl get svc -n jenkins
kubectl get pvc -n jenkins
```

Access Jenkins:

```text
http://<node-ip>:32000
```

---

### 5) RBAC for Jenkins Kubernetes Deployments

#### 5.1 Create ServiceAccount

```bash
kubectl -n jenkins create sa jenkins-sa
```

#### 5.2 ClusterRoleBinding

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-sa-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-sa
  namespace: jenkins
EOF
```

Since Jenkins controller pod used SA `jenkins`, binding applied:

```bash
kubectl create clusterrolebinding jenkins-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=jenkins:jenkins
```

Verify:

```bash
kubectl get pod jenkins-0 -n jenkins -o jsonpath='{.spec.serviceAccountName}{"
"}'
```

---

### 6) AWS ECR Repository Setup

Create repo:

```bash
aws ecr create-repository --repository-name demo-nginx --region ap-south-1
```

ECR URI:

```text
231907690017.dkr.ecr.ap-south-1.amazonaws.com/demo-nginx
```

---

### 7) Configure AWS Credentials in Jenkins

Jenkins UI:

* **Manage Jenkins → Credentials → System → Global → Add Credentials**

Credential:

* Type: **Username with password**
* ID: `aws-creds`
* Username: `AWS_ACCESS_KEY_ID`
* Password: `AWS_SECRET_ACCESS_KEY`

---

### 8) Jenkins Pipeline from GitHub (SCM Integration)

Create Jenkins job:

* **New Item → Pipeline**

Configure:

* Definition: ✅ **Pipeline script from SCM**
* SCM: Git
* Repository URL:

  ```text
  https://github.com/BhushanKhutle/jenkins-ecr-k8s-demo.git
  ```
* Branch: `*/main`
* Script Path: `Jenkinsfile`

✅ Build executed successfully.

---

### 9) GitHub Webhook Integration

#### 9.1 Enable trigger in Jenkins job

In Jenkins Job → Configure → Build Triggers:

* ✅ GitHub hook trigger for GITScm polling

#### 9.2 Add webhook in GitHub repo

Payload URL used:

```text
http://k8stesting-647846595.ap-south-1.elb.amazonaws.com/github-webhook/
```

Webhook config:

* Content type: `application/json`
* Events: `push`

✅ Every Git push triggers Jenkins pipeline automatically.

---

### 10) Validate Deployment

Check deployment:

```bash
kubectl get deploy,pods,svc -n demo -o wide
```

Test app:

```bash
curl http://<worker-node-ip>:<nodeport>
```

---

## 🖼 Screenshots

Add screenshots here (recommended for portfolio):

* ✅ Jenkins Dashboard
* ✅ Pipeline successful build
* ✅ AWS ECR image pushed
* ✅ Kubernetes deployment + service
* ✅ Application output in browser/curl

Example folder structure:

```text
screenshots/
├── 01-jenkins-dashboard.png
├── 02-pipeline-success.png
├── 03-ecr-image.png
├── 04-k8s-deployment.png
└── 05-app-output.png
```

---

## 🧯 Troubleshooting

### 1) Jenkins PVC stuck in Pending

Cause: No StorageClass.
Fix: Install NFS dynamic provisioner.

Check:

```bash
kubectl get pvc -n jenkins
kubectl get sc
```

---

### 2) Jenkins Helm install fails with adminUser error

Error:

* `controller.adminUser no longer exists`

Fix:
Use:

```yaml
controller:
  admin:
    username: admin
    password: "Jenkins@123"
```

---

### 3) Jenkins pipeline fails: `checkout scm` not available

Cause: Job is **Pipeline script**, not SCM-based.
Fix:

* Use **Pipeline script from SCM**

---

### 4) Kubernetes Agent pod `sh` not found

Cause: Minimal images don’t contain `/bin/sh`.
Fix:
Use stable tool image:

* `dtzar/helm-kubectl:3.15.4`

---

## 🚀 Future Enhancements

* ✅ Deploy using **Helm chart** instead of kubectl create
* ✅ Blue/Green or Canary Deployments
* ✅ Add Rollback stage in pipeline
* ✅ Add Prometheus + Grafana monitoring dashboard for app
* ✅ Add centralized logging using Loki + Promtail
* ✅ Configure TLS with cert-manager
* ✅ Implement least privilege RBAC for Jenkins

---

## 🧾 Resume Bullet Points

* Deployed Jenkins on kubeadm Kubernetes cluster using Helm with persistent volumes via NFS StorageClass.
* Implemented end-to-end CI/CD pipeline using Jenkins Kubernetes agent pods.
* Integrated GitHub webhook to trigger Jenkins builds automatically.
* Built and pushed Docker images to AWS ECR with versioned tags.
* Automated Kubernetes deployments with rollout verification and zero manual deployment steps.
