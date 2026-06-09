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

## Развертывание сайта в кластере Kubernetes

### 1. Добавьте образ Django Site контейнера в кластер

Для этого заранее создайте образ, чтобы он отображался в Docker Desktop во вкладке Images. Затем в запущенном кластере запустите команду для загрузки образа в кластер:

```bash
minikube image load <имя_образа>:<версия>
```

Команда для просмотра загруженных в кластер образов:

```bash
minikube image ls
```

### 2. Установка PostgreSQL в кластер

- Для быстрой установки БД мы будем использовать Helm. И для начала его нужно скачать с официального (репозитория)[https://github.com/helm/helm/releases/tag/v4.2.0], выберите нужную версию для вашей системы, скачайте и добавьте его в окружение вашей системы. Команда для проверки, что установка прошла успешно:

```bash
helm version
```

- Следующим шагом создайте манифест-файл для развертывания БД, в котором будут перечислены ее параметры. Файл должен быть такого вида:

```yaml
auth:
  postgresPassword: "password"
  username: "test_k8s"
  password: "password"
  database: "test_k8s"

containerPorts:
  postgresql: 5432

primary:
  service:
    ports:
      postgresql: 5432
  persistence:
    enabled: true
    size: 8Gi
```

Вам нужно заменить `password` на свой пароль. Затем сохраните файл в директорию `kubernetes`, перейдите в эту директорию внутри вашей командой строки и используйте команду:

```bash
kubectl apply -f <имя_файла.yaml>
```

### 3. Создание Secrets в кластере

- Создайте манифест-файл для Secrets, по которому сможете развернуть их в кластере. Образец yaml файла:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
stringData:
  database-url: "url"
  secret-key: "secret_key" 
```

- Замените `url` на фактический адрес к вашей БД следующей формы: `postgresql://<имя_пользователя>:<пароль>@<хост>:5432/<имя_БД>`

- Придумайте и вставьте свой `secret_key` любого формата

- Сохраните файл и, перейдя в его директорию в командой строке, используйте следующую команду для создания Secrets в кластере:

```bash
kubectl apply -f <имя_файла.yaml>
```

### 4. Создание Django Site Deployment

В директории `kubernetes` вы можете найти манифест-файл для развертывания `deployment` в кластере, в котором описаны основыне параметры будущего `deployment` объекта.

- Внутри командной строки перейдите в директорию `kubernetes` и разверните деплоймент с помощью следующей комнады:

```bash
kubectl apply -f <имя_файла.yaml>
```

### 5. Настройка Ingress

Чтобы не открывать каждый сервис через NodePort или LoadBalancer в кластере создается `Ingress`, который распределяет трафик внутри кластера по нужным сервисам.

- Установите nginx командой:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/cloud/deploy.yaml
```

- Создайте ingress-hosts.yaml файл по типу:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-hosts

spec:
  ingressClassName: nginx
  rules:
  - host: star-burger.test
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: django-site
            port:
              number: 80
```

- Запустите в кластере команду создания Ingress по новому манифесту:

```bash
kubectl apply -f ingress-hosts.yaml
```

- После этого откройте дополнительное окно с командой строкой и создайте маршруты для сервисов командой:

```bash
minikube tunnel
```

Дополнительная команда, для посмотра запущенных `ingress`:

```bash
kubectl get ingress
```

### 6. Настройка регулярного удаления сессий

- Для регулярного удаления старых сессий пользователей нужно запустить `CronJob` из манифеста `clearsessions-v2.yaml` командой:

```bash
kubectl apply -f .../clearsessions-v2.yaml
```

- В манифест файле с помощью параметра `schedule` можно гибко настроить периодичность выполнения задачи. Подбробнее о [Cron syntax](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) можно прочитать в документации.

- Для принудительного выполнения задачи можно воспользоваться командой:

```bash
kubectl create job --from=cronjob/<имя_cronjob> <придумайте_имя_job>
```