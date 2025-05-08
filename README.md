# ZopDevAssessment
# MinIO Distributed Deployment on Kubernetes using Helm

This repository contains instructions and configuration for deploying [MinIO](https://min.io) in **distributed mode** on **Kubernetes** using **Helm**. The deployment supports:

- Distributed mode with multiple pods for high availability
- Secure access using TLS (optional)
- User access via LoadBalancer or Ingress
- Secure storage of credentials using Kubernetes Secrets

---

## Prerequisites

Before you begin, ensure the following are installed:

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm v3](https://helm.sh/docs/intro/install/)
- A running Kubernetes cluster (e.g., Minikube, GKE, EKS, AKS, etc.)

---

## Step 1: Create Kubernetes Secret for Access Credentials

MinIO requires an `access key` and a `secret key`. Store them securely using Kubernetes Secrets.

### Encode Your Keys:

```bash
echo -n "minio-access-key" | base64
echo -n "minio-secret-key" | base64
```

### Create `minio-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-creds
type: Opaque
data:
  accesskey: <base64-encoded-access-key>
  secretkey: <base64-encoded-secret-key>
```

### Apply the Secret:

```bash
kubectl apply -f minio-secrets.yaml
```

---

## Step 2: Set Up TLS

### Generate a Self-Signed Certificate:

```bash
openssl genpkey -algorithm RSA -out minio.key
openssl req -new -key minio.key -out minio.csr -subj "/CN=minio.assessment.com"
openssl x509 -req -in minio.csr -signkey minio.key -out minio.crt
```

### Create TLS Secret:

```bash
kubectl create secret tls minio-tls-secret --cert=minio.crt --key=minio.key
```

---

## Step 3: Prepare `values.yaml`

Create a `values.yaml` file to customize your deployment.

### `values.yaml`:

```yaml
mode: distributed
replicas: 4

existingSecret: minio-creds

resources:
  requests:
    memory: 512Mi
    cpu: 250m

persistence:
  enabled: true
  size: 10Gi

service:
  type: LoadBalancer

ingress:
  enabled: false  # Set to true if you want to expose via Ingress
  annotations: {}
  hosts:
    - minio.assessment.com

tls:
  enabled: true
  certSecret: minio-tls-secret

consoleService:
  type: LoadBalancer
```

> For Ingress, configure `ingress.enabled: true` and set host rules.

---

## Step 4: Deploy MinIO with Helm

### Add MinIO Helm Repo:

```bash
helm repo add minio https://charts.min.io/
helm repo update
```

### Install MinIO:

```bash
helm install minio minio/minio --values values.yaml
```

---

## Updating or Reinstalling MinIO

### Uninstall:

```bash
helm uninstall minio
```

### Reinstall:

```bash
helm install minio minio/minio --values values.yaml
```

---

## Accessing MinIO

After installation:

```bash
kubectl get svc
```

Use the external IP or hostname to access MinIO:

- Console: `https://<EXTERNAL-IP>:9001`
- API: `https://<EXTERNAL-IP>:9000`

If using Ingress, access it via the configured domain (e.g., `https://minio.assessment.com`).

---

## Notes

- For production, consider using valid certificates via Let's Encrypt and cert-manager.
- Ensure storage backends (PVCs) are correctly provisioned.
- Configure Ingress and TLS termination securely based on your platform (Nginx, Traefik, etc.).

---

## Cleanup

```bash
helm uninstall minio
kubectl delete secret minio-creds
kubectl delete secret minio-tls-secret
```

---
