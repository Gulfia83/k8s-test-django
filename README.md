# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Развёртывание сайта в Minikube

### Установка postgreSQL в кластер
1. Установите [Helm](https://helm.sh/) для вашей ОС
2. Установите postgreSQL с помощью [Helm](https://artifacthub.io/packages/helm/bitnami/postgresql):
    ```bash
    helm install <postgres_pod_name> \
    --set auth.username=<your_username> \
    --set auth.password=<your_password> \
    --set auth.database=<your_db_name> \
    oci://registry-1.docker.io/bitnamicharts/postgresql
    ```
3. Проверьте корректность установки с помощью:
    ```bash
    kubectl get pod  # в списке должен быть под с postgreSQL
    kubectl get svc  # в списке должен быть как минимум один сервис, связанный с postgreSQL
    ```
4. Для подключения Django к СУБД используйте в качестве имени хоста имя Pod postgreSQL.
    ```bash
    ❯ kubectl get pod
    NAME                         READY   STATUS    RESTARTS   AGE
    django-app-db-postgresql-0   1/1     Running   0          61s

    ```
    В данном случае `DATABASE_URL` должен быть следующего вида: `postgres://<your_username>:<your_password>@django-app-db-postgresql.default.svc.cluster.local:5432/<your_db_name>`
### Добавление переменных окружения

1. Создайте `YAML` файл c чувствительными переменными:

@    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: django-secrets
    type: Opaque
    data:
      SECRET_KEY: 'base64_secret_key'
      DATABASE_URL: 'base64_database_url'
      ALLOWED_HOSTS: 'base64_allowed_hosts'
    ```

Все значения переменных должны быть закодированы в `BASE64`.

2. Создайте сущностm `Secret` в вашем кластере:
    ```bash
    kubectl apply -f path/to/django-app-secret.yml

    ```

### Создание Ingress
1. Пропишите свою конфигурацию в YAML файл
  
2. Подключите аддон ingress для вашего кластера:
    ```bash
    minikube addons enable ingress
    ```
3. Создайте сущность `Ingress` в вашем кластере
    ```bash
    kubectl apply -f path/to/django-app-ingress.yml
    ```
4. (опционально) Пропишите, если нужно, ваш хост в файле hosts вашей ОС:
    ```hosts
    <minikube_ip> your_host
    ```

### Создание Deployment
1. Загрузите ваш Docker Image в Docker Hub
2. Пропишите свою конфигурацию в YAML файл
    
3. Создайте сущность `Deployment` в кластере:
    ```bash
    kubectl apply -f path/to/django-app-deployment.yml
    ```
### Применение миграций
1. Пропишите свою конфигурацию в YAML файл
  
2. Создайте сущность Job. После создания она сразу же запустится и применит миграции.
    ```bash
    kubectl apply -f path/to/django-app-migrate.yml
    ```
3. Для отслеживания статуса Job используйте:
    ```bash
    kubectl get job
    ```
    Она должна иметь статус `Completed`.
### Создание superuser
1. Получите список подов с помощью:
    ```bash
    kubectl get pods
    ```
2. Выполните вход в любой из подов, принадлежащих deployment с джанго:
    ```bash
    kubectl exec -it <django_pod_name> -- /bin/bash
    ```
3. Создайте superuser с помощью:
    ```bash
    python manage.py createsuperuser
    ```
4. Выйдите из пода:
    ```bash
    exit
    ```
### Создание Service
1. Пропишите свою конфигурацию в YAML файл:

2. Создайте сущность Service с помощью:
    ```bash
    kubectl apply -f path/to/django-app-service.yml
    ```

### Регулярная очистка сессий с помощью CronJob
1. Пропишите свою конфигурацию в YAML файл:

2. Создайте сущность CronJob с помощью команды:
    ```bash
    kubectl apply -f path/to/django-app-clearsessions.yml
    ```
    Эта задача будет производить очистку сессий каждую неделю.

## Развёртывание dev-версии в кластере Yandex Cloud

### Подготовка к развёртыванию
1. Получите доступ к кластеру у его администратора.
2. Установите и инициализируйте [Yandex Cloud CLI](https://yandex.cloud/ru/docs/cli/quickstart#install).
3. Добавьте учётные данные кластера в конфигурационный файл kubectl:
    ```bash
    yc managed-kubernetes cluster get-credentials --id cat528346gdueh53ts39 --external
    ```
### Запуск nginx
1. Создайте YAML конфигурацию для `Pod`
   
2. Создайте сущность `Pod`
    ```bash
    kubectl apply -f path/to/nginx-pod.yml
    ```
3. Создайте YAML конфигурацию для `Service`
   
4. Создайте сущность `Service`
    ```bash
    kubectl apply -f path/to/nginx-service.yml
    ```
5. Nginx будет доступен по вашему домену.

### Подключение к PostgreSQL
1. Проверьте, присутствует ли в вашем `namespace` секрет с ssl сертификатом для PostgreSQL. Если присутствует, переходите сразу к п.4. Если нет, то получите ваш ssl сертификат [по этой инструкции](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect#get-ssl-cert).
2. Создайте YAML конфигурацию для `Secret` с сертификатом:
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: pg-root-cert
    data:
      root.crt: <base64_encoded_ssl_certificate>
    ```
3. Создайте сущность `Secret` с помощью:
    ```bash
    kubectl apply -f path/to/secret.yaml
    ```
4. Создайте YAML конфигурацию для `Deployment` пода с Ubuntu, с помощью которого вы будете подключаться к СУБД.
    
6. Создайте сущность `Deployment` с помощью:
    ```bash
    kubectl apply -f path/to/ubuntu-deployment.yml
    ```
7. Зайдите в shell контейнера с помощью:
    ```bash
    kubectl exec -it <pod_name> -- /bin/bash
    ```
8. Подключитесь к СУБД:
    ```bash
    psql "host=<postgres_host> port=<postgres_port> dbname=<db_name> user=<user_name> password=<user_password>"
    ```

### Добавление переменных окружения

1. Создайте `YAML` файл c чувствительными переменными:

@    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: django-secrets
    type: Opaque
    data:
      SECRET_KEY: 'base64_secret_key'
      DATABASE_URL: 'base64_database_url'
      ALLOWED_HOSTS: 'base64_allowed_hosts'
    ```

Все значения переменных должны быть закодированы в `BASE64`.

2. Создайте сущностm `Secret` в вашем кластере:
    ```bash
    kubectl apply -f path/to/django-secret.yml

    ```

### Создание Deployment
1. Загрузите ваш Docker Image в Docker Hub
2. Пропишите свою конфигурацию в YAML файл
    
3. Создайте сущность `Deployment` в кластере:
    ```bash
    kubectl apply -f path/to/django-deployment.yml
    ```
### Применение миграций
1. Пропишите свою конфигурацию в YAML файл
  
2. Создайте сущность Job. После создания она сразу же запустится и применит миграции.
    ```bash
    kubectl apply -f path/to/django-migrate.yml
    ```
3. Для отслеживания статуса Job используйте:
    ```bash
    kubectl get job
    ```
    Она должна иметь статус `Completed`.
### Создание superuser
1. Получите список подов с помощью:
    ```bash
    kubectl get pods
    ```
2. Выполните вход в любой из подов, принадлежащих deployment с джанго:
    ```bash
    kubectl exec -it <django_pod_name> -- /bin/bash
    ```
3. Создайте superuser с помощью:
    ```bash
    python manage.py createsuperuser
    ```
4. Выйдите из пода:
    ```bash
    exit
    ```
### Создание Service
1. Пропишите свою конфигурацию в YAML файл:

2. Создайте сущность Service с помощью:
    ```bash
    kubectl apply -f path/to/django-service.yml
    ```

### Регулярная очистка сессий с помощью CronJob
1. Пропишите свою конфигурацию в YAML файл:

2. Создайте сущность CronJob с помощью команды:
    ```bash
    kubectl apply -f path/to/django-clearsessions.yml
    ```
    Эта задача будет производить очистку сессий каждую неделю.

[Пример сайта](https://edu-determined-johnson.sirius-k8s.dvmn.org/)

***
Код написан в учебных целях — это урок в курсе по Python и веб-разработке на сайте [Devman](https://dvmn.org).
