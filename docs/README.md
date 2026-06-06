# Лабораторная работа по Kubernetes - Messenger

## Структура репозитория

```
k8s/
├── base/                          ← общие манифесты для всех окружений
│   ├── kustomization.yaml
│   ├── namespace.yaml             ← namespace: lab-kuber
│   ├── configmap.yaml             ← несекретные переменные (порты, URL)
│   ├── secret.yaml                ← пароли БД
│   ├── csi-s3-secret.yaml         ← пароли MinIO
│   ├── pg-pvc.yaml                ← хранилище для postgres
│   ├── pg-service.yaml            ← service для postgres
│   ├── postgres.yaml              ← БД + init-скрипт создания messager_users/messager_messages
│   ├── minio.yaml                 ← локальный S3 (affinity: workload=system)
│   ├── pv-pvc-minio.yaml          ← PV/PVC через CSI S3-driver
│   ├── migrate-users.yaml         ← goose миграции для users DB
│   ├── migrate-messages.yaml      ← goose миграции для messages DB
│   ├── users-service.yaml         ← порт 8081, affinity: workload=app
│   ├── messages-service.yaml      ← порт 8082, affinity: workload=app (hard) + disk=fast (soft)
│   ├── bff.yaml                   ← порт 8080, affinity: workload=app
│   ├── frontend.yaml              ← порт 80, affinity: workload=app
│   └── ingress.yaml               ← nginx ingress
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml     ← 1 реплика, dev.messager.local, latest теги
│   │   └── patches/
│   │       ├── dev-resources.yaml
│   │       └── dev-images.yaml
│   └── prod/
│       ├── kustomization.yaml     ← 2 реплики, prod хост, теги по digest
│       └── patches/
│           ├── prod-resources.yaml
│           └── prod-images.yaml
argocd/
├── application-dev.yaml           ← Argo CD app → k8s/overlays/dev
└── application-prod.yaml          ← Argo CD app → k8s/overlays/prod
docs/
├── Отчёт по Kubernetes.docx       ← отчёт
└── README.md                      ← этот файл
```

## Порты сервисов

| Сервис          | Порт |
|-----------------|------|
| frontend        | 80   |
| bff             | 8080 |
| user-service    | 8081 |
| message-service | 8082 |
| postgres        | 5432 |
| minio (S3 API)  | 9000 |
| minio (console) | 9001 |

## Инструкция по запуску

**Замечание**: предполагается, что kubectl и minikube уже установлены

Запуск minikube и установка Ingress
```bash
minikube start --nodes 4 --cpus=2 --memory=3784 --driver=docker
minikube addons enable ingress
```

Выдача нодам лейблов
```bash
kubectl label node minikube-m02 workload=system
kubectl label node minikube-m03 workload=app
kubectl label node minikube-m04 workload=app disk=fast
```

Создание драйвера CSI
```bash
cd ~
git clone https://github.com/yandex-cloud/k8s-csi-s3.git
cd k8s-csi-s3/deploy/kubernetes
kubectl apply -f provisioner.yaml
kubectl apply -f driver.yaml
kubectl apply -f csi-s3.yaml
```

Создание namespace'a
```bash
kubectl apply -f k8s/base/namespace.yaml
```

Создание minio и bucket'a в нём
```bash
cd k8s/base
kubectl apply -f csi-s3-secret.yaml
kubectl apply -f pv-pvc-minio.yaml
kubectl apply -f minio.yaml

kubectl port-forward -n lab-kuber svc/minio 9000:9000 &

curl -LO https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && sudo mv mc /usr/local/bin/

mc alias set local http://localhost:9000 minioadmin minioadmin678
mc mb local/uploads
```

Установка ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Подключение к ArgoCD
```bash
#пароль
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
#подключение к argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
#применение application
kubectl apply -f argocd/application-dev.yaml
```

Запуск базового кластера:
```bash
kubectl apply -f k8s/base
```

Проверка корректности работы
```bash
# Cборка overlays
kubectl kustomize k8s/overlays/dev
kubectl kustomize k8s/overlays/prod

# Статус подов
kubectl get pods -n lab-kuber

# nodeAffinity
kubectl get pods -o wide -n lab-kuber
kubectl describe pod <имя-пода-message-service> -n lab-kuber | grep -A 10 Affinity

# Монтирование S3
kubectl exec -n lab-kuber deployment/message-service -- df -h /app/uploads

# Статус Argo CD
kubectl get application -n argocd
```

Ожидаемые статусы подов:
- postgres, minio, frontend, bff, user-service, message-service → `Running`
- migrate-users, migrate-messages → `Completed`
- Argo CD → `Synced / Healthy`