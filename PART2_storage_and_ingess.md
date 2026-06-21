## Проблема которую мы решаем

Демонстрация проблемы с данными

Прежде чем чинить — убеждаемся, что проблема реальна. Заходим в PostgreSQL и создаем тестовые данные:
```
kubectl exec -it -n workshop $(kubectl get pod -n workshop -l app=postgres -o jsonpath='{.items[0].metadata.name}') -- psql -U workshop -d workshop_db -c "CREATE TABLE test (id serial, note text); INSERT INTO test (note) VALUES ('переживу ли я рестарт?');"
```
Проверяем, что данные на месте:
```
kubectl exec -it -n workshop $(kubectl get pod -n workshop -l app=postgres -o jsonpath='{.items[0].metadata.name}') -- psql -U workshop -d workshop_db -c "SELECT * FROM test;"
```
Убиваем ПОД, деплоймент пересоздает его автоматичски:
```
kubectl delete pod -n workshop -l app=postgres
```
Проверяем данные:
```
kubectl exec -it -n workshop $(kubectl get pod -n workshop -l app=postgres -o jsonpath='{.items[0].metadata.name}') -- psql -U workshop -d workshop_db -c "SELECT * FROM test;"
```
Получаем вывод:
```
ERROR:  relation "test" does not exist
```
![part2_problem_test.png](screenshots/part2_problem_test.png)

Почему так происходит: Контейнер пишет данные в собственную файловую систему, которая существует только пока живёт контейнер. Когда Pod пересоздаётся — создаётся новый контейнер с чистой файловой системой. Это поведение Docker, и Kubernetes его не меняет — он просто оркестрирует контейнеры, а не данные внутри них.

## Теория. Три уовня абстракции

1. PersisentVolume(PV) - Реальный кусок диска. Существует на уровне всего кластера.
2. PersistentVolumeClaim(PVC) - Запрос на место. Существует в namespace.
3. StorageClass - Шаблон по которому PV создается автоматически, по запросу PVC

Под не обращается к PV напрямую - только через PVC. PVC. Это разделение позволяет администратору управлять реальным диском (NFS, AWS EBS, GCP Disk).

Я создаю PV вручную специально, чтобы увидеть механику изнутри, прежде чем доверять её автоматике.

## Подготовка директории на ноде

PersistentVolume в нашем случае будет указывать на реальную папку на диске Minikube-ноды
```
minikube ssh "sudo mkdir -p /mnt/data/postgres"
```

## Создаем PersistenVolume

[09_postgres_pv.yaml](manifests/09_postgres_pv.yaml)

```
1. kubectl apply -f manifests/09_postgres_pv.yaml
2. kubectl get pv
```
![part2_get_pv](screenshots/part2_get_pv.png)

## Создаем PersistentVolumeClaim

[10_postgres_pvc.yaml](manifests/10_postgres_pvc.yaml)

```
1. kubectl apply -f manifests/10_postgres_pvc.yaml
2. kubectl get pvc -n workshop
```

STATUS: Bound — это ключевой момент. Kubernetes нашёл PV, который подходит под запрос PVC (тот же storageClassName, достаточный размер, совпадающий accessMode), и связал их. Если бы подходящего PV не было — статус остался бы Pending.

[part2_pvc.png](screenshots/part2_pvc.png)

## Подключаем PVC к Deployment PostgreSQL

[04_db_deloyment.yaml](manifests/04_db_deployment.yaml)

```
1. kubectl apply -f manifests/04_db_deployment.yaml
2. kubectl get pods -n workshop -w
```
