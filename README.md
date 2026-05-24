# Курсовий проєкт: Docker & Kubernetes

## Опис проєкту

У цьому проєкті розгортається self-hosted Git-сервіс **Gitea** у Kubernetes кластері.

Проєкт реалізований за GitOps-підходом з використанням **FluxCD**. Уся інфраструктура описана як код у Git-репозиторії. Будь-яка зміна в репозиторії автоматично синхронізується з Kubernetes-кластером без ручного `kubectl apply`.

---

# Технології

- Docker
- Kubernetes
- Helm
- FluxCD
- CloudNativePG Operator
- PostgreSQL
- Gitea
- NGINX Ingress Controller
- HPA

---

# Архітектура

Застосунок: **Gitea**  
База даних: **PostgreSQL**  
Оператор бази даних: **CloudNativePG**

| Environment | Namespace | Replicas | Database | Ingress |
|---|---|---|---|---|
| staging | staging | 1 | PostgreSQL single instance | gitea.staging.local |
| production | production | HPA 2-5 | PostgreSQL production config | gitea.local |

---

# 1. Встановлення Kubernetes-кластера

## Встановлення необхідних інструментів

### Docker

```bash
docker --version
Docker version 29.3.0-1, build 5927d80c76b3ce5cf782be818922966e8a0d87a3
```

### kubectl

```bash
kubectl version --client
Client Version: v1.35.2
Kustomize Version: v5.7.1
```

### Helm

```bash
helm version
version.BuildInfo{Version:"v4.1.1", GitCommit:"5caf0044d4ef3d62a955440272999e139aafbbed", GitTreeState:"clean", GoVersion:"go1.25.7", KubeClientVersion:"v1.35"}
```

### Flux CLI

```bash
flux --version
flux version 2.8.8
```

### kind

```bash
kind version
kind v0.29.0 go1.24.2 linux/amd64
```

---

# 2. Створення Kubernetes-кластера

Створюємо файл `kind-cluster.yaml`

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: course-project
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

Створюємо кластер:

```bash
kind create cluster --config kind-cluster.yaml
Creating cluster "course-project" ...
 ✓ Ensuring node image (kindest/node:v1.33.1) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-course-project"
You can now use your cluster with:

kubectl cluster-info --context kind-course-project

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

Перевіряємо:

```bash
kubectl get nodes
NAME                           STATUS     ROLES           AGE   VERSION
course-project-control-plane   NotReady   control-plane   23s   v1.33.1
```

---

# 3. Встановлення Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
serviceaccount/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
configmap/ingress-nginx-controller created
service/ingress-nginx-controller created
service/ingress-nginx-controller-admission created
deployment.apps/ingress-nginx-controller created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
ingressclass.networking.k8s.io/nginx created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
```

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

```bash
kubectl get pods -n ingress-nginx
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7b887fdf8-z4m9c   1/1     Running   0          47s
```

---

# 4. Підготовка локальних доменів

```bash
sudo sh -c 'echo "127.0.0.1 gitea.staging.local" >> /etc/hosts'
sudo sh -c 'echo "127.0.0.1 gitea.local" >> /etc/hosts'
```

---

# 5. Структура репозиторію

```text
course-project/
├── README.md
├── kind-cluster.yaml
├── charts/
│   └── gitea-app/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── secret.yaml
│           └── hpa.yaml
├── infrastructure/
│   ├── operators/
│   │   └── cloudnative-pg/
│   │       ├── namespace.yaml
│   │       ├── helmrepository.yaml
│   │       ├── helmrelease.yaml
│   │       └── kustomization.yaml
│   └── databases/
│       ├── staging-postgres.yaml
│       ├── production-postgres.yaml
│       └── kustomization.yaml
└── clusters/
    ├── staging/
    │   ├── namespace.yaml
    │   ├── helmrelease.yaml
    │   └── kustomization.yaml
    └── production/
        ├── namespace.yaml
        ├── helmrelease.yaml
        └── kustomization.yaml
```

---

# 6. Ініціалізація FluxCD

```bash
flux bootstrap github \
  --owner=YevhenMarholin \
  --repository=course-project-Yevhen-Marholin \
  --branch=main \
  --path=./clusters \
  --personal
► connecting to github.com
► cloning branch "main" from Git repository "https://github.com/YevhenMarholin/course-project-Yevhen-Marholin.git"
✔ cloned repository
► generating component manifests
✔ generated component manifests
✔ component manifests are up to date
► installing components in "flux-system" namespace
✔ installed components
✔ reconciled components
► determining if source secret "flux-system/flux-system" exists
► generating source secret
✔ public key: ecdsa-sha2-nistp384 AAAAE2VjZHNhLXNoYTItbmlzdHAzODQAAAAIbmlzdHAzODQAAABhBHGlgj3+ef3PzgDbpKBdInw2neym9L3auu8h4eZgkbhX6aEi5E0zVjN/NRh89ocdPKOaLUf1pEvxWiPiZG2B/Vxd7b2eKVltkSsobicqsz+BA3H8IoR9Iy6izYIuVl0a3A==
✔ configured deploy key "flux-system-main-flux-system-./clusters" for "https://github.com/YevhenMarholin/course-project-Yevhen-Marholin"
► applying source secret "flux-system/flux-system"
✔ reconciled source secret
► generating sync manifests
✔ generated sync manifests
✔ committed sync manifests to "main" ("9d210a0758c41188f57057f1c83b2c25816207c6")
► pushing sync manifests to "https://github.com/YevhenMarholin/course-project-Yevhen-Marholin.git"
► applying sync manifests
✔ reconciled sync configuration
◎ waiting for GitRepository "flux-system/flux-system" to be reconciled
✔ GitRepository reconciled successfully
◎ waiting for Kustomization "flux-system/flux-system" to be reconciled
✔ Kustomization reconciled successfully
► confirming components are healthy
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all components are healthy
```

```bash
flux check
```

---

# 7. CloudNativePG Operator

## namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cnpg-system
```

## helmrepository.yaml

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: cloudnative-pg
  namespace: flux-system
spec:
  interval: 1h
  url: https://cloudnative-pg.github.io/charts
```

## helmrelease.yaml

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cloudnative-pg
  namespace: flux-system
spec:
  interval: 10m
  targetNamespace: cnpg-system
  chart:
    spec:
      chart: cloudnative-pg
      version: "*"
      sourceRef:
        kind: HelmRepository
        name: cloudnative-pg
        namespace: flux-system
      interval: 1h
```

## kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
```

---

# 8. PostgreSQL через Operator

## staging-postgres.yaml

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: gitea-postgres
  namespace: staging
spec:
  instances: 1

  storage:
    size: 1Gi

  bootstrap:
    initdb:
      database: gitea
      owner: gitea
      secret:
        name: gitea-db-secret
```

## production-postgres.yaml

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: gitea-postgres
  namespace: production
spec:
  instances: 1

  storage:
    size: 2Gi

  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 500m

  bootstrap:
    initdb:
      database: gitea
      owner: gitea
      secret:
        name: gitea-db-secret
```

---

# 9. Helm Chart для Gitea

## Chart.yaml

```yaml
apiVersion: v2
name: gitea-app
description: Gitea application Helm chart for course project
type: application
version: 0.1.0
appVersion: "1.22"
```

## values.yaml

```yaml
replicaCount: 1

image:
  repository: gitea/gitea
  tag: "1.22"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 3000

ingress:
  enabled: true
  className: nginx
  host: gitea.local

resources: {}

database:
  host: gitea-postgres-rw
  port: 5432
  name: gitea
  user: gitea
  password: gitea-password

hpa:
  enabled: false
  minReplicas: 2
  maxReplicas: 5
  cpuUtilization: 70
```

---

# 10. Перевірка роботи

## HelmRelease

```bash
flux get helmreleases -A
```

## Kustomizations

```bash
flux get kustomizations -A
```

## Pods

```bash
kubectl get pods -A
```

## Ingress

```bash
kubectl get ingress -A
```

## HPA

```bash
kubectl get hpa -A
```

---

# 11. Self-Healing перевірка

```bash
kubectl delete deployment gitea -n staging
```

```bash
kubectl get deployment -n staging
```

```bash
flux reconcile kustomization staging --with-source
```

---

# 12. Результат роботи

Було реалізовано:

- Kubernetes cluster
- FluxCD GitOps
- CloudNativePG Operator
- PostgreSQL через CRD
- Gitea application
- Helm chart
- staging environment
- production environment
- HPA
- ingress
- self-healing