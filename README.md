# Portal Demo - Kubernetes CI/CD Lab

## 專案目標

建立一套完整的 CI/CD 流程：

GitHub → Jenkins → Harbor → Helm → K3s

當程式碼推送到 GitHub 時：

1. Jenkins 自動觸發
2. Build Docker Image
3. Push Image 到 Harbor
4. 使用 Helm 更新 Kubernetes Deployment
5. K3s 自動 Rolling Update

---

# 架構圖

```text
Developer
    │
    ▼
 GitHub
    │
    ▼
 Jenkins
    │
    ├── docker build
    ├── docker push
    ▼
 Harbor
    │
    ▼
 Helm Upgrade
    │
    ▼
 K3s Cluster
    │
    ▼
 Rolling Update
```

---

# 環境

## Host

Ubuntu Linux

## Container Runtime

Docker

## Kubernetes

K3s

## CI

Jenkins

## Registry

Harbor

## Source Control

GitHub

---

# Harbor 安裝

下載 Harbor Offline Installer：

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.14.0/harbor-offline-installer-v2.14.0.tgz
```

解壓縮：

```bash
tar xvf harbor-offline-installer-v2.14.0.tgz
```

修改：

```bash
harbor.yml
```

設定：

```yaml
hostname: 172.17.69.81

http:
  port: 8088
```

產生設定：

```bash
sudo ./prepare
```

啟動：

```bash
sudo docker compose up -d
```

驗證：

```bash
docker ps
```

登入：

```text
http://172.17.69.81:8088
```

建立 Project：

```text
portal
```

---

# Docker Image

Build：

```bash
docker build -t portal:v1 .
```

Tag：

```bash
docker tag portal:v1 \
172.17.69.81:8088/portal/portal:v1
```

Push：

```bash
docker push \
172.17.69.81:8088/portal/portal:v1
```

---

# K3s

確認：

```bash
kubectl get nodes
```

建立 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: portal

spec:
  replicas: 2

  selector:
    matchLabels:
      app: portal

  template:
    metadata:
      labels:
        app: portal

    spec:
      containers:
      - name: portal
        image: portal:v1
```

建立 Service：

```yaml
apiVersion: v1
kind: Service

metadata:
  name: portal-service

spec:
  selector:
    app: portal

  ports:
  - port: 80
    targetPort: 5000
```

---

# Harbor Authentication

建立 Harbor Secret：

```bash
kubectl create secret docker-registry harbor-secret \
  --docker-server=172.17.69.81:8088 \
  --docker-username=admin \
  --docker-password=<password>
```

如果 Secret 已存在：

```bash
kubectl delete secret harbor-secret
```

重新建立：

```bash
kubectl create secret docker-registry harbor-secret ...
```

Deployment 加入：

```yaml
imagePullSecrets:
- name: harbor-secret
```

---

# 問題排查

## ErrImagePull

```text
pull access denied
no basic auth credentials
```

原因：

沒有 Harbor Secret。

解法：

```yaml
imagePullSecrets:
- name: harbor-secret
```

---

## ErrImageNeverPull

```text
ErrImageNeverPull
```

原因：

```yaml
imagePullPolicy: Never
```

解法：

```yaml
imagePullPolicy: Always
```

---

# Jenkins

啟動：

```bash
docker start jenkins
```

驗證：

```bash
curl http://localhost:8081/login
```

---

# GitHub Repository

```bash
git init

git remote add origin \
git@github.com:liuallen871219/portal-demo.git

git add .
git commit -m "initial commit"

git push -u origin master
```

---

# Jenkins Pipeline from SCM

設定：

```text
Pipeline script from SCM
```

Repository：

```text
git@github.com:liuallen871219/portal-demo.git
```

Script Path：

```text
Jenkinsfile
```

---

# Jenkinsfile

```groovy
pipeline {

    agent any

    options {
        buildDiscarder(
            logRotator(
                numToKeepStr: '20'
            )
        )
    }

    stages {

        stage('Build') {
            steps {
                sh '''
                docker build \
                  -t 172.17.69.81:8088/portal/portal:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push') {
            steps {
                sh '''
                docker push \
                  172.17.69.81:8088/portal/portal:${BUILD_NUMBER}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                helm upgrade --install portal helm/portal \
                  --set image.tag=${BUILD_NUMBER}

                kubectl rollout status deployment/portal
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                kubectl get pods -o wide
                kubectl get svc
                '''
            }
        }
    }
}
```

---

# Helm

建立 Chart：

```bash
helm create portal
```

結構：

```text
helm/
└── portal/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

---

# values.yaml

```yaml
image:
  repository: 172.17.69.81:8088/portal/portal
  tag: latest
```

---

# deployment.yaml

```yaml
spec:
  imagePullSecrets:
  - name: harbor-secret

  containers:
  - name: portal
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    imagePullPolicy: Always
```

---

# 驗證

查看 Release：

```bash
helm list
```

查看 Revision：

```bash
helm history portal
```

查看 Manifest：

```bash
helm get manifest portal
```

---

# 最終成果

完成：

* GitHub Source Control
* Jenkins CI
* Harbor Registry
* Docker Build
* Docker Push
* K3s Cluster
* Kubernetes Deployment
* Service
* Ingress
* Harbor Secret
* Helm Chart
* Helm Upgrade
* Rolling Update
* SCM Auto Trigger

---

# 未來改進

## GitHub Webhook

取代 Poll SCM：

```text
GitHub Push
     │
     ▼
Webhook
     │
     ▼
Jenkins
```

---

## ArgoCD

升級為 GitOps：

```text
GitHub
   │
   ▼
Jenkins
   │
   ▼
Harbor
   │
   ▼
ArgoCD
   │
   ▼
K3s
```

---

## Monitoring

導入：

* Prometheus
* Grafana

---

## Logging

導入：

* Loki
* Promtail

---

## Security

導入：

* Trivy Image Scan
* Harbor Vulnerability Scan
* RBAC

```
```

