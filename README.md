# 📘 End-to-End Project Documentation

## Jenkins on Kubernetes (kubeadm) + GitHub Webhook + AWS ECR + Auto Deployment

**Author:** Bhushan Khutle (DevOps Engineer)

---

## 1. Project Overview

This project implements a complete DevOps CI/CD pipeline:

* ✅ Jenkins installed on Kubernetes (kubeadm cluster) using Helm
* ✅ Jenkins persistent storage enabled using NFS Dynamic Provisioner (StorageClass)
* ✅ Jenkins pipelines run using Kubernetes Agent Pods
* ✅ GitHub push triggers Jenkins automatically using Webhook
* ✅ Jenkins builds Docker images and pushes to AWS ECR
* ✅ Jenkins deploys/updates application on Kubernetes automatically
* ✅ App is reachable using Kubernetes Service and Ingress/ELB

---

## 2. Architecture Flow

```
Developer Push → GitHub Repo
        ↓ (Webhook)
Jenkins (Running on Kubernetes)
        ↓
Pipeline runs in Kubernetes Agent Pod
        ↓
Docker Build
        ↓
Push Image to AWS ECR
        ↓
Deploy to Kubernetes (demo namespace)
        ↓
Expose using Service/Ingress
```

---

## 3. Technologies Used

### Kubernetes / DevOps

* Kubernetes cluster created using **kubeadm**
* Calico CNI
* Nginx Ingress Controller
* Jenkins deployed on Kubernetes using Helm
* Jenkins Kubernetes Agent Pods

### AWS

* AWS ECR (Elastic Container Registry)

### SCM

* GitHub repository + webhook

---

## 4. Kubernetes Cluster Pre-check

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get sc
```

---

## 5. Jenkins Installation on Kubernetes

### 5.1 Create Jenkins Namespace

```bash
kubectl create ns jenkins
```

### 5.2 Install Helm (if required)

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 6. Storage Setup (No StorageClass Fix)

The cluster initially had **no StorageClass**, so Jenkins PVC could not bind.

To fix this, NFS-based dynamic provisioning was configured.

---

## 7. NFS StorageClass Setup

### 7.1 Setup NFS Server (RHEL)

Install packages:

```bash
yum install -y nfs-utils rpcbind
systemctl enable --now rpcbind
systemctl enable --now nfs-server
```

Create export directory:

```bash
mkdir -p /mnt/k8s-nfs
chmod -R 777 /mnt/k8s-nfs
```

Configure NFS export:

```bash
cat <<EOF > /etc/exports
/mnt/k8s-nfs *(rw,sync,no_root_squash,no_subtree_check)
EOF
```

Apply export:

```bash
exportfs -rav
showmount -e localhost
```

---

### 7.2 Install NFS Provisioner (Dynamic PV Provisioning)

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

Install provisioner:

```bash
helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  -n kube-system \
  --set nfs.server=<NFS_SERVER_IP> \
  --set nfs.path=/mnt/k8s-nfs
```

Verify:

```bash
kubectl get pods -n kube-system | grep nfs
kubectl get sc
```

---

## 8. Install Jenkins using Helm

### 8.1 Add Jenkins Helm Repo

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

### 8.2 Create `values.yaml`

> Note: `controller.adminUser` is deprecated. New format is `controller.admin.username`.

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

### 8.3 Install Jenkins

```bash
helm install jenkins jenkins/jenkins -n jenkins -f values.yaml
```

Verify:

```bash
kubectl get pods -n jenkins -w
kubectl get svc -n jenkins
kubectl get pvc -n jenkins
```

### 8.4 Access Jenkins UI

```text
http://<node-ip>:32000
```

Login:

* Username: `admin`
* Password: `Jenkins@123`

---

## 9. Jenkins Kubernetes Integration (RBAC)

### 9.1 Create ServiceAccount

```bash
kubectl -n jenkins create sa jenkins-sa
```

### 9.2 ClusterRoleBinding (Admin access for pipelines)

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

Since Jenkins controller pod was using service account `jenkins`, cluster-admin was applied:

```bash
kubectl create clusterrolebinding jenkins-admin \
  --clusterrole=cluster-admin \
  --serviceaccount=jenkins:jenkins
```

Verify Jenkins pod SA:

```bash
kubectl get pod jenkins-0 -n jenkins -o jsonpath='{.spec.serviceAccountName}{"\n"}'
```

---

## 10. AWS ECR Setup

### 10.1 Create ECR Repository

```bash
aws ecr create-repository --repository-name demo-nginx --region ap-south-1
```

ECR URI:

```text
231907690017.dkr.ecr.ap-south-1.amazonaws.com/demo-nginx
```

---

## 11. Add AWS Credentials in Jenkins

Jenkins UI:
**Manage Jenkins → Credentials → System → Global → Add Credentials**

* Type: `Username with password`
* ID: `aws-creds`
* Username: `AWS_ACCESS_KEY_ID`
* Password: `AWS_SECRET_ACCESS_KEY`

---

## 12. GitHub Repository

Repository:

```text
https://github.com/BhushanKhutle/jenkins-ecr-k8s-demo.git
```

Repo structure:

```
jenkins-ecr-k8s-demo/
├── Dockerfile
├── index.html
└── Jenkinsfile
```

---

## 13. Jenkins Pipeline from SCM (GitHub Integration)

### 13.1 Create Jenkins Job

* New Item → Pipeline

### 13.2 Configure Pipeline

In Jenkins job:

* Definition: ✅ Pipeline script from SCM
* SCM: Git
* Repo URL: `https://github.com/BhushanKhutle/jenkins-ecr-k8s-demo.git`
* Branch: `*/main`
* Script path: `Jenkinsfile`

Pipeline build status: ✅ SUCCESS

---

## 14. GitHub Webhook Integration

### 14.1 Webhook URL

Payload URL used:

```text
http://k8stesting-647846595.ap-south-1.elb.amazonaws.com/github-webhook/
```

### 14.2 Jenkins Trigger

In Jenkins job config:
✅ Enable:

* GitHub hook trigger for GITScm polling

Result:
✅ Every push to GitHub triggers Jenkins pipeline automatically.

---

## 15. Final Deployment Verification

Check resources:

```bash
kubectl get deploy,pods,svc -n demo -o wide
```

Test app access using NodePort:

```bash
curl http://<worker-node-ip>:<nodeport>
```

---

## 16. Final Outcome

✅ Jenkins installed and operational on Kubernetes
✅ Persistent storage configured using NFS provisioner
✅ Kubernetes agent pods executing CI/CD pipelines
✅ Docker images pushed to AWS ECR
✅ Kubernetes deployment updated automatically
✅ GitHub webhook triggers full pipeline

---

## 17. Resume / Interview Bullet Points

* Implemented end-to-end CI/CD pipeline using Jenkins deployed on a kubeadm Kubernetes cluster.
* Configured Jenkins persistent storage using NFS Dynamic Provisioner (StorageClass).
* Integrated GitHub webhook to trigger Jenkins builds automatically.
* Built Docker images and pushed versioned tags to AWS ECR.
* Automated Kubernetes deployments with rollout verification.
