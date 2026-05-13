# Мой первый проект k8s

## Цель проекта 
Задеплоить полноценное приложение с базой данных, настроить конфиги, секреты и масштабирование

## Структура проекта
```
~/projects/k8s_workshop
 ├── manifests
 │   ├── 01_namespace.yaml
 │   ├── 02_configmap.yaml
 │   ├── 03_secret.yaml
 │   ├── 04_db_deployment.yaml
 │   ├── 05_db_service.yaml
 │   ├── 06_app_deployment.yaml
 │   ├── 07_app_service.yaml
 │   └── 08_app_deployment_v2.yaml
 └── README.md
```

## Установка kubectl и minikube

### kubectl

1. url -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

2. sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

### minikube

1. curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
2 .sudo install minikube-linux-amd64 /usr/local/bin/minikube

## Запуск minikube и ingress addon

1. minikube start --driver=docker --cpus=2 --memory=4096

minicube создает k8s кластер внутри docker контейнера

2. minikube addons enable ingress

По умолчанию Minikube не умеет маршрутизировать HTTP-трафик снаружи. Ingress-аддон устанавливает nginx-контроллер, который будет принимать запросы и направлять их к нужным сервисам внутри кластера.

## Namespace

[01_namespace.yaml](manifests/01_namespace.yaml)

### Применяем

1. kubectl apply -f manifests/01_namespace.yaml

Тем самым создаем отдельное окружение внутри кластера, что бы не смешивать их с компонентами kube-system

что бы не прописывать -n workshop устанавливаем namespace по умолчанию

2. kubectl config set-context --current --namespace=workshop


## Configmap

[02_configmap.yaml](manifests/02_configmap.yaml)

### Запускаем

1. kubectl apply -f manifests/02_configmap.yaml

Что бы не пересобирать docker image при каждом изменении настройки, отделяем конфигурацию от образа контейнера

## Secrets

[03_secret.yaml](manifests/03_secret.yaml)

### Применяем

1. kubectl apply -f manifests/03_secret.yaml

## База данных

[04_db_deployment.yaml](manifests/04_db_deployment.yaml)

### Применяем

1. kubectl apply -f manifests/04_db_deployment.yaml

Зачем каждый блок:

1. selector.matchLabels — связывает Deployment с Pod'ами. Deployment управляет только теми Pod'ами, у которых совпадают labels.

2. env.valueFrom.secretKeyRef — внедряет значение из Secret как переменную окружения. Контейнер получает реальное значение, а не base64.

3. resources.requests — минимальные гарантированные ресурсы. Scheduler использует их при выборе ноды.

4. resources.limits — жёсткий потолок. Превышение RAM = OOMKill. Превышение CPU = троттлинг (не убивает, но замедляет).

5. readinessProbe — пока PostgreSQL запускается, Pod не получает трафик. Защита от запросов к "ещё не готовой" БД.

6. livenessProbe — если PostgreSQL завис, K8s перезапустит контейнер. Авто-хилинг.

## Service для PostgreSQL

[05_db_service.yaml](manifests/05_db_service.yaml)

### Применяем

1. kubectl apply -f manifests/05_db_service.yaml

Pod'ы эфемерны — они перезапускаются и получают новый IP при каждом рестарте. Service — это стабильный DNS-адрес, который всегда указывает на живые Pod'ы. Внутри кластера можно обращаться к PostgreSQL просто по имени postgres-service:5432. Kubernetes DNS автоматически разрешает это имя в нужный IP.





