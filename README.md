# Курсовий проєкт: Docker & Kubernetes

## Опис проєкту

У цьому проєкті розгортається self-hosted Git-сервіс **Gitea** у Kubernetes кластері.

Проєкт реалізований за GitOps-підходом з використанням **FluxCD**. Уся інфраструктура описана як код у Git-репозиторії. Будь-яка зміна в репозиторії автоматично синхронізується з Kubernetes-кластером без ручного `kubectl apply`.

---

## Технології

- Docker
- Kubernetes
- Helm
- FluxCD
- CloudNativePG Operator
- PostgreSQL
- Gitea
- NGINX Ingress Controller
- HPA
- kind

---

## Архітектура

Застосунок: **Gitea**  
База даних: **PostgreSQL**  
Оператор бази даних: **CloudNativePG**

| Environment | Namespace | Replicas | Database | Ingress |
|---|---|---:|---|---|
| staging | staging | 1 | PostgreSQL single instance | gitea.staging.local |
| production | production | HPA 1-2 | PostgreSQL production config | gitea.local |

---

## 1. Встановлення необхідних інструментів

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

## 2. Створення Kubernetes-кластера

Файл `kind-cluster.yaml`:

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

Створення кластера:

```bash
kind create cluster --config kind-cluster.yaml
```

Результат:

```text
Creating cluster "course-project" ...
 ✓ Ensuring node image (kindest/node:v1.33.1)
 ✓ Preparing nodes
 ✓ Writing configuration
 ✓ Starting control-plane
 ✓ Installing CNI
 ✓ Installing StorageClass
Set kubectl context to "kind-course-project"
```

Перевірка:

```bash
kubectl get nodes
```

```text
NAME                           STATUS   ROLES           AGE   VERSION
course-project-control-plane   Ready    control-plane   40m   v1.33.1
```

---

## 3. Встановлення Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Очікування готовності:

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

Перевірка:

```bash
kubectl get pods -n ingress-nginx
```

```text
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7b887fdf8-z4m9c   1/1     Running   0          39m
```

---

## 4. Підготовка локальних доменів

```bash
sudo sh -c 'echo "127.0.0.1 gitea.staging.local" >> /etc/hosts'
sudo sh -c 'echo "127.0.0.1 gitea.local" >> /etc/hosts'
```

У GitHub Codespaces доступ до застосунку також перевірявся через port-forward:

```bash
kubectl port-forward svc/gitea -n staging 3000:3000 --address 0.0.0.0
kubectl port-forward svc/gitea -n production 3001:3000 --address 0.0.0.0
```

---

## 5. Структура репозиторію

```text
course-project-Yevhen-Marholin/
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
│           └── hpa.yaml
├── infrastructure/
│   ├── operators/
│   │   └── cloudnative-pg/
│   │       ├── namespace.yaml
│   │       ├── helmrepository.yaml
│   │       ├── helmrelease.yaml
│   │       └── kustomization.yaml
│   └── databases/
│       ├── staging/
│       │   ├── postgres.yaml
│       │   └── kustomization.yaml
│       └── production/
│           ├── postgres.yaml
│           └── kustomization.yaml
└── clusters/
    ├── kustomization.yaml
    ├── flux-system/
    │   ├── gotk-components.yaml
    │   ├── gotk-sync.yaml
    │   └── kustomization.yaml
    ├── staging/
    │   ├── namespace.yaml
    │   ├── db-secret.yaml
    │   ├── helmrelease.yaml
    │   └── kustomization.yaml
    └── production/
        ├── namespace.yaml
        ├── db-secret.yaml
        ├── helmrelease.yaml
        └── kustomization.yaml
```

---

## 6. Ініціалізація FluxCD

```bash
flux bootstrap github \
  --owner=YevhenMarholin \
  --repository=course-project-Yevhen-Marholin \
  --branch=main \
  --path=./clusters \
  --personal
```

Результат:

```text
✔ cloned repository
✔ generated component manifests
✔ installed components
✔ reconciled components
✔ configured deploy key
✔ reconciled sync configuration
✔ GitRepository reconciled successfully
✔ Kustomization reconciled successfully
✔ all components are healthy
```

Перевірка Flux:

```bash
flux check
```

```text
✔ Kubernetes 1.33.1 >=1.33.0-0
✔ distribution: flux-v2.8.8
✔ bootstrapped: true
✔ helm-controller: deployment ready
✔ kustomize-controller: deployment ready
✔ notification-controller: deployment ready
✔ source-controller: deployment ready
✔ all checks passed
```

---

## 7. CloudNativePG Operator

CloudNativePG встановлюється через Flux `HelmRelease`.

### infrastructure/operators/cloudnative-pg/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cnpg-system
```

### infrastructure/operators/cloudnative-pg/helmrepository.yaml

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

### infrastructure/operators/cloudnative-pg/helmrelease.yaml

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

### infrastructure/operators/cloudnative-pg/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
```

### clusters/flux-system/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - gotk-components.yaml
  - gotk-sync.yaml
  - ../../infrastructure/operators/cloudnative-pg
```

---

## 8. PostgreSQL через CloudNativePG Operator

Використання plain-text StatefulSet або Deployment для бази даних не застосовується. PostgreSQL створюється через Custom Resource `Cluster`, який надає CloudNativePG Operator.

### clusters/staging/db-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitea-db-secret
  namespace: staging
type: kubernetes.io/basic-auth
stringData:
  username: gitea
  password: gitea-password
```

### clusters/production/db-secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitea-db-secret
  namespace: production
type: kubernetes.io/basic-auth
stringData:
  username: gitea
  password: gitea-password
```

### infrastructure/databases/staging/postgres.yaml

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

### infrastructure/databases/production/postgres.yaml

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
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 300m

  bootstrap:
    initdb:
      database: gitea
      owner: gitea
      secret:
        name: gitea-db-secret
```

### infrastructure/databases/staging/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - postgres.yaml
```

### infrastructure/databases/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - postgres.yaml
```

---

## 9. Helm Chart для Gitea

### charts/gitea-app/Chart.yaml

```yaml
apiVersion: v2
name: gitea-app
description: Gitea application Helm chart for course project
type: application
version: 0.1.0
appVersion: "1.22"
```

### charts/gitea-app/values.yaml

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
  minReplicas: 1
  maxReplicas: 2
  cpuUtilization: 70
```

### charts/gitea-app/templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    app: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: gitea
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 3000
          env:
            - name: GITEA__database__DB_TYPE
              value: postgres
            - name: GITEA__database__HOST
              value: "{{ .Values.database.host }}:{{ .Values.database.port }}"
            - name: GITEA__database__NAME
              value: {{ .Values.database.name | quote }}
            - name: GITEA__database__USER
              valueFrom:
                secretKeyRef:
                  name: gitea-db-secret
                  key: username
            - name: GITEA__database__PASSWD
              valueFrom:
                secretKeyRef:
                  name: gitea-db-secret
                  key: password
          resources:
{{ toYaml .Values.resources | indent 12 }}
```

### charts/gitea-app/templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app: {{ .Release.Name }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: 3000
```

### charts/gitea-app/templates/ingress.yaml

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ .Release.Name }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ .Release.Name }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

### charts/gitea-app/templates/hpa.yaml

```yaml
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.cpuUtilization }}
{{- end }}
```

---

## 10. Flux environments

### clusters/staging/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
```

### clusters/staging/helmrelease.yaml

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: gitea
  namespace: staging
spec:
  interval: 5m
  chart:
    spec:
      chart: ./charts/gitea-app
      sourceRef:
        kind: GitRepository
        name: flux-system
        namespace: flux-system
  values:
    replicaCount: 1
    ingress:
      host: gitea.staging.local
    resources: {}
    hpa:
      enabled: false
```

### clusters/staging/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - db-secret.yaml
  - ../../infrastructure/databases/staging
  - helmrelease.yaml
```

### clusters/production/namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

### clusters/production/helmrelease.yaml

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: gitea
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: ./charts/gitea-app
      sourceRef:
        kind: GitRepository
        name: flux-system
        namespace: flux-system
  values:
    replicaCount: 1

    ingress:
      host: gitea.local

    resources:
      requests:
        cpu: 50m
        memory: 128Mi
      limits:
        cpu: 100m
        memory: 256Mi

    hpa:
      enabled: true
      minReplicas: 1
      maxReplicas: 2
      cpuUtilization: 70
```

### clusters/production/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - db-secret.yaml
  - ../../infrastructure/databases/production
  - helmrelease.yaml
```

### clusters/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - flux-system
  - staging
  - production
```

---

## 11. GitOps sync

Після створення або зміни файлів потрібно виконати commit та push:

```bash
git add .
git commit -m "Add Gitea application with PostgreSQL operator"
git push origin main
```

Примусова синхронізація Flux:

```bash
flux reconcile kustomization flux-system -n flux-system --with-source
```

Результат:

```text
✔ fetched revision main@sha1:4d102e8eb8adbee89f499471ab7d3c3bb82841bc
✔ applied revision main@sha1:4d102e8eb8adbee89f499471ab7d3c3bb82841bc
```

---

## 12. Перевірка роботи

### HelmRelease

```bash
flux get helmreleases -A
```

Результат:

```text
NAMESPACE     NAME             AGE   READY   STATUS
flux-system   cloudnative-pg   17m   True    Helm install succeeded for release cnpg-system/cnpg-system-cloudnative-pg.v1 with chart cloudnative-pg@0.28.2
production    gitea            10s   True    Helm install succeeded for release production/gitea.v1 with chart gitea-app@0.1.0
staging       gitea            34m   True    Helm install succeeded for release staging/gitea.v1 with chart gitea-app@0.1.0
```

### Kustomizations

```bash
flux get kustomizations -A
```

Результат:

```text
NAMESPACE     NAME          REVISION           SUSPENDED   READY   MESSAGE
flux-system   flux-system   main@sha1:4d102e8e False       True    Applied revision: main@sha1:4d102e8e
```

### Pods

```bash
kubectl get pods -A
```

Результат:

```text
NAMESPACE            NAME                                                   READY   STATUS    RESTARTS   AGE
cnpg-system          cnpg-system-cloudnative-pg-6ccf6b4fd8-hcd4m            1/1     Running   0          17m
flux-system          helm-controller-7bd48c8dfc-rfv2m                       1/1     Running   0          60m
flux-system          kustomize-controller-86b794fcbf-96rtf                  1/1     Running   0          60m
flux-system          notification-controller-76bb5947d4-zxbtf               1/1     Running   0          60m
flux-system          source-controller-5fb95cbb75-ttsgk                     1/1     Running   0          60m
ingress-nginx        ingress-nginx-controller-7b887fdf8-z4m9c               1/1     Running   0          94m
production           gitea-6b74c46d99-qwpjz                                 1/1     Running   0          1m
production           gitea-postgres-1                                       1/1     Running   0          1m
staging              gitea-cd6dbcb55-gzbmx                                  1/1     Running   0          1m
staging              gitea-postgres-1                                       1/1     Running   0          25m
```

### PostgreSQL clusters

```bash
kubectl get clusters.postgresql.cnpg.io -A
```

Результат:

```text
NAMESPACE    NAME             AGE   INSTANCES   READY   STATUS                     PRIMARY
production   gitea-postgres   20m   1           1       Cluster in healthy state   gitea-postgres-1
staging      gitea-postgres   20m   1           1       Cluster in healthy state   gitea-postgres-1
```

### Ingress

```bash
kubectl get ingress -A
```

Результат:

```text
NAMESPACE    NAME    CLASS   HOSTS                 ADDRESS     PORTS   AGE
production   gitea   nginx   gitea.local           localhost   80      3m
staging      gitea   nginx   gitea.staging.local   localhost   80      35m
```

### HPA

```bash
kubectl get hpa -A
```

Результат:

```text
NAMESPACE    NAME    REFERENCE          TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
production   gitea   Deployment/gitea   cpu: <unknown>/70%   1         2         1          1m
```

У локальному kind-кластері `TARGETS` може бути `<unknown>`, якщо не встановлено `metrics-server`. Сам HPA об'єкт створений і працює як Kubernetes ресурс.

---

## 13. Перевірка доступу до застосунку

### Staging

```bash
kubectl port-forward svc/gitea -n staging 3000:3000 --address 0.0.0.0
```

Після цього застосунок відкривається через forwarded port у GitHub Codespaces.

Також перевірка через `curl`:

```bash
curl http://gitea.staging.local
```

Результат:

```html
<!DOCTYPE html>
<html lang="en-US">
<title>Installation - Gitea: Git with a cup of tea</title>
```

### Production

```bash
kubectl port-forward svc/gitea -n production 3001:3000 --address 0.0.0.0
```

Після цього застосунок відкривається через forwarded port у GitHub Codespaces.

Також перевірка через `curl`:

```bash
curl http://gitea.local
```

Результат:

```html
<!DOCTYPE html>
<html lang="en-US">
<title>Installation - Gitea: Git with a cup of tea</title>
```
![alt text](image.png)
![alt text](image-1.png)
![alt text](image-2.png)
---

## 14. Self-Healing перевірка

Видаляємо deployment вручну:

```bash
kubectl delete deployment gitea -n staging
```

Примусово запускаємо reconciliation HelmRelease:

```bash
flux reconcile helmrelease gitea -n staging --force
```

Після reconciliation deployment автоматично відновлюється:

```bash
kubectl get deployment -n staging
```

Результат:

```text
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
gitea   1/1     1            1           10s
```

Також pod застосунку створюється заново:

```bash
kubectl get pods -n staging
```

Результат:

```text
NAME                    READY   STATUS    RESTARTS   AGE
gitea-cd6dbcb55-gzbmx   1/1     Running   0          10s
gitea-postgres-1        1/1     Running   0          25m
```

---

## 15. Результат роботи

У результаті було реалізовано:

- створення Kubernetes кластера через kind
- встановлення NGINX Ingress Controller
- ініціалізація FluxCD
- GitOps-підхід до доставки інфраструктури
- встановлення CloudNativePG Operator через Flux HelmRelease
- створення PostgreSQL бази даних через Custom Resource
- створення власного Helm chart для Gitea
- staging середовище
- production середовище
- Ingress для обох середовищ
- HPA для production
- self-healing через FluxCD
![alt text](image-2.png)