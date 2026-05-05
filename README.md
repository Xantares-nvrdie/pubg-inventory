# 📘 RUNBOOK LENGKAP: AKS + Kubernetes + Jenkins + CI/CD

---

# 🧩 0. PRASYARAT

Install:

```bash
brew install azure-cli kubectl docker ngrok
```

Login:

```bash
az login
```

---

# ☁️ 1. BUAT AKS (Azure Kubernetes)

## 🔹 Buat Resource Group

```bash
az group create --name rg-pubgm --location southeastasia
```

## 🔹 Buat Cluster AKS

```bash
az aks create \
  --resource-group rg-pubgm \
  --name pubgm-cluster \
  --node-count 1 \
  --enable-addons monitoring \
  --generate-ssh-keys
```

---

# 🔑 2. AMBIL KUBE CONFIG (AKSES CLUSTER)

```bash
az aks get-credentials \
  --resource-group rg-pubgm \
  --name pubgm-cluster
```

➡️ ini akan merge ke:

```text
~/.kube/config
```

---

# 🧪 3. TEST KONEKSI

```bash
kubectl get nodes
```

---

# 🌐 4. INSTALL INGRESS CONTROLLER

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

---

## 🔹 Cek service ingress

```bash
kubectl get svc -n ingress-nginx
```

➡️ ambil:

```text
EXTERNAL-IP
```

---

# 📦 5. BUAT FILE KUBERNETES

---

## 🔹 5.1 Deployment + Service (pubgm-k8s.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-pubgm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend-api
        image: x4ntares/pubgm-backend:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        env:
        - name: NAMA_PRAKTIKAN
          value: "Nama Kamu"
        - name: NIM_PRAKTIKAN
          value: "123456"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 5000
      targetPort: 5000
  type: ClusterIP
```

---

## 🔹 5.2 Frontend

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-pubgm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend-ui
        image: x4ntares/pubgm-frontend:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

---

## 🔹 5.3 Ingress (pubgm-ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pubgm-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 5000
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

---

# 🚀 6. DEPLOY KE AKS

## 🔹 Buat namespace

```bash
kubectl create namespace pubgm
```

## 🔹 Apply semua

```bash
kubectl apply -f pubgm-k8s.yaml -n pubgm
kubectl apply -f pubgm-ingress.yaml -n pubgm
```

---

## 🔹 Cek pod

```bash
kubectl get pods -n pubgm
```

---

## 🔹 Cek ingress

```bash
kubectl get ingress -n pubgm
```

---

# 🐳 7. BUILD & PUSH DOCKER

## 🔹 Login Docker

```bash
docker login
```

## 🔹 Build multi-arch

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  -t x4ntares/pubgm-backend:v1 ./backend --push

docker buildx build --platform linux/amd64,linux/arm64 \
  -t x4ntares/pubgm-frontend:v1 ./frontend --push
```

---

# 🔁 8. UPDATE IMAGE DI KUBERNETES

```bash
kubectl set image deployment/backend-pubgm \
  backend-api=x4ntares/pubgm-backend:v2 -n pubgm

kubectl set image deployment/frontend-pubgm \
  frontend-ui=x4ntares/pubgm-frontend:v2 -n pubgm
```

---

# 🔍 9. DEBUGGING

## 🔹 cek log

```bash
kubectl logs deployment/backend-pubgm -n pubgm
```

## 🔹 cek env

```bash
kubectl exec -it deployment/backend-pubgm -n pubgm -- printenv
```

---

# 🤖 10. SETUP JENKINS

## 🔹 Install Jenkins

```bash
brew install jenkins-lts
brew services start jenkins-lts
```

Buka:

```text
http://localhost:8080
```

---

## 🔹 Tambah credential

* DockerHub (username/password)
* kubeconfig (secret file)

---

# 🔁 11. PIPELINE (Jenkinsfile)

```groovy
pipeline {
  agent any

  environment {
    DOCKER_HUB_USER = 'x4ntares'
    BUILD_TAG = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Build & Push') {
      steps {
        sh '''
        docker buildx build --platform linux/amd64,linux/arm64 \
          -t x4ntares/pubgm-backend:${BUILD_TAG} ./backend --push
        '''
      }
    }

    stage('Deploy') {
      steps {
        sh '''
        kubectl apply -f pubgm-k8s.yaml -n pubgm
        '''
      }
    }
  }
}
```

---

# 🔗 12. WEBHOOK (AUTO TRIGGER)

## 🔹 Jalankan ngrok

```bash
ngrok http 8080
```

## 🔹 Tambahkan ke GitHub

```text
https://xxxx.ngrok-free.app/github-webhook/
```

---

# 🧪 13. TEST

```bash
curl http://<EXTERNAL-IP>/api/info
```

---

# 🎯 ALUR FINAL

```text
git push
 ↓
webhook
 ↓
jenkins
 ↓
docker build
 ↓
push docker hub
 ↓
deploy AKS
 ↓
web update
```

---

# 🧠 CATATAN PENTING

* `kubectl apply` → update config
* `kubectl set image` → update image
* ENV hanya update via `apply`
* Ingress hanya routing HTTP
* Service hanya networking internal

---

# 🏁 SELESAI

Kamu sekarang punya:
✔ AKS
✔ Kubernetes
✔ CI/CD
✔ Web public

---
