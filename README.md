# Мой первый проект k8s

## Цель проекта. Часть 1
Задеплоить полноценное приложение с базой данных, настроить конфиги, секреты и масштабирование

## Структура проекта
```
k8s_workshop/
 ├── PART1_base_deployment.md
 ├── manifests
 │   ├── 01_namespace.yaml
 │   ├── 02_configmap.yaml
 │   ├── 03_secret.yaml
 │   ├── 04_db_deployment.yaml
 │   ├── 05_db_service.yaml
 │   ├── 06_app_deployment.yaml
 │   ├── 07_app_service.yaml
 │   └── 08_app_deployment_v2.yaml
 └─── README.md
```
## Цель проекта. Часть 2
Продолжение базового workshop. Здесь мы решаем две проблемы текущего проекта:
  1. PostgreSQL теряет данные при перезапуске Pod
  2. Доступ к приложению идёт через неудобный NodePort вместо нормального домена

## Структура проекта
```
k8s-workshop/
│── PART2_storage_and_ingress.md
|── manifests/
│   ├── 01_namespace.yaml
│   ├── 02_configmap.yaml
│   ├── 03_secret.yaml
│   ├── 04_db_deployment.yaml      ← обновлён: добавлен volumeMounts
│   ├── 05_db_service.yaml
│   ├── 06_app_deployment.yaml
│   ├── 07_app_service.yaml
│   ├── 08_app_deployment_v2.yaml
│   ├── 09_postgres_pv.yaml        ← новый
│   ├── 10_postgres_pvc.yaml       ← новый
│   └── 11_app_ingress.yaml        ← новый
└─── README.md
