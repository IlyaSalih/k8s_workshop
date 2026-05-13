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

## Deployment для приложений

[06_app_deployment.yaml](manifests/06_app_deployment.yaml)

1. kubectl apply -f manifests/06-app-deployment.yaml

rollingUpdate: при maxUnavailable: 0 и maxSurge: 1 обновление выглядит так:

1. Создаётся новый Pod (итого: 3 Pod)
2. Новый Pod становится Ready
3. Один старый Pod удаляется (итого: 2 Pod)
4. Процесс повторяется

Трафик никогда не прерывается — пользователи не замечают деплой.

volumes: ConfigMap монтируется как директория. Ключ nginx.conf превращается в файл /etc/nginx/conf.d/default.conf. Это позволяет изменить конфиг nginx без пересборки образа — достаточно обновить ConfigMap.

## Service для приложений

[07_app_service.yaml](manifests/07_app_service.yaml)

### Применяем

1. kubectl apply -f manifests/07_app_service.yaml

NodePort вместо ClusterIP: ClusterIP работает только внутри кластера. NodePort открывает порт на каждой ноде кластера (диапазон 30000–32767), делая сервис доступным снаружи. Для Minikube это самый простой способ получить доступ к приложению. 

## Проверка и отладка

### Проверка состояния всего

1. kubectl get all 

```
NAME                                       READY   STATUS    RESTARTS   AGE
pod/app-deployment-6db5769449-hj4n2        1/1     Running   0          25m
pod/app-deployment-6db5769449-hshm2        1/1     Running   0          25m
pod/postgres-deployment-7894c9b8b6-9zwhn   1/1     Running   0          10d

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/app-service        NodePort    10.108.225.193   <none>        80:32206/TCP   7m34s
service/postgres-service   ClusterIP   10.107.105.124   <none>        5432/TCP       56m

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-deployment        2/2     2            2           25m
deployment.apps/postgres-deployment   1/1     1            1           10d

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/app-deployment-6db5769449        2         2         2       25m
replicaset.apps/postgres-deployment-589c8688b8   0         0         0       10d
replicaset.apps/postgres-deployment-7894c9b8b6   1         1         1       10d
```

### Открываем приложение в браузере

2. minikube service app-service -n workshop

```
┌───────────┬─────────────┬─────────────┬───────────────────────────┐
│ NAMESPACE │    NAME     │ TARGET PORT │            URL            │
├───────────┼─────────────┼─────────────┼───────────────────────────┤
│ workshop  │ app-service │ 80          │ http://192.168.49.2:32206 │
└───────────┴─────────────┴─────────────┴───────────────────────────┘
🎉  Opening service workshop/app-service in default browser...
```

![in browser](screenshots/in%20browser)

minikube автоматически пробрасывает порт и открывает браузер, где задеплоин nginx

## Обновление версии приложения

[08_app_deployment.yaml](manifests/08_app_deployment_v2.yaml)

1. kubectl apply -f manifests/08_app_deployment_v2.yaml

### Проверка обновления

1. kubectl get pods -n workshop

```
NAME                                   READY   STATUS    RESTARTS   AGE
app-deployment-746d989459-kq2lw        1/1     Running   0          11m
app-deployment-746d989459-r7mlg        1/1     Running   0          12m
postgres-deployment-7894c9b8b6-9zwhn   1/1     Running   0          10d
```

2. kubectl describe pod -n workshop -l app=web-app | grep Image:
```
    Image:          nginx:1.27-alpine
    Image:          nginx:1.27-alpine
```

3. kubectl rollout history deployment/app-deployment -n workshop
```
deployment.apps/app-deployment 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

4. kubectl rollout undo deployment/app-deployment -n workshop

5. kubectl get pods -n workshop
```
NAME                                   READY   STATUS    RESTARTS   AGE
app-deployment-6db5769449-g7gkf        1/1     Running   0          33s
app-deployment-6db5769449-hzw76        0/1     Running   0          10s
app-deployment-746d989459-kq2lw        1/1     Running   0          16m
postgres-deployment-7894c9b8b6-9zwhn   1/1     Running   0          10d
```

6. kubectl describe pod -n workshop -l app=web-app | grep Image:
```
    Image:          nginx:1.25-alpine
    Image:          nginx:1.27-alpine
    Image:          nginx:1.27-alpine
```

7. kubectl rollout history deployment/app-deployment -n workshop
```
deployment.apps/app-deployment 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```






