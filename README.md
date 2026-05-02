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
```
url -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
### minikube
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
## Запуск minikube и ingress addon
```
minikube start --driver=docker --cpus=2 --memory=4096

minicube создает k8s кластер внутри docker контейнера

minikube addons enable ingress

По умолчанию Minikube не умеет маршрутизировать HTTP-трафик снаружи. Ingress-аддон устанавливает nginx-контроллер, который будет принимать запросы и направлять их к нужным сервисам внутри кластера.
```
## Задаем namespace
```
.
 ├── manifests
 |   ├── 01_namespace.yaml
```
```
# yaml

apiVersion: v1
kind: Namespace
metadata:
  name: workshop
  labels:
    environment: learning
```
```
# Применяем

kubectl apply -f manifests/01-namespace.yaml

Тем самым создаем отдельное окружение внутри кластера, что бы не смешивать их с компонентами kube-system

что бы не прописывать -n workshop устанавливаем namespace по умолчанию

kubectl config set-context --current --namespace=workshop
```

## Configmap
```
 ├── manifests
 |   ├── 02_configmap.yaml
```
```
# yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: workshop
data:
  # Переменные окружения для приложения
  APP_ENV: "production"
  APP_PORT: "8080"
  DB_HOST: "postgres-service"   # имя Service базы данных
  DB_PORT: "5432"
  DB_NAME: "workshop_db"
  # Конфигурационный файл целиком (multi-line)
  nginx.conf: |
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /health {
            return 200 'OK';
            add_header Content-Type text/plain;
        }
    }
```
```
# Запускаем

kubectl apply -f manifests/02-configmap.yaml

Что бы не пересобирать docker image при каждом изменении настройки, отделяем конфигурацию от образа контейнера
```






