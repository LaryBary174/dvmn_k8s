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

# Minikube
Необходимо по инструкции установить [VirtualBox](https://www.virtualbox.org/wiki/Downloads)
Скачать файлы и внести их расположение в системные переменные среды env ОС
[minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)
[kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
[helm](https://github.com/helm/helm/releases)

## Запуск minikube
Команда для запуска :
```
minikube start --driver=virtualbox --no-vtx-check
```
## Docker Image 
Сборка docker images по dockerfile в данном проекте

Для использования Docker напрямую в контексте Minikube необходимо выполнить команду:
```
minikube -p minikube docker-env --shell powershell | Invoke-Expression

```
Дальше переходим в директорию Dockerfile и производим сборку image данного приложения:
```
docker build -t django-app:latest .

```
## Postgresql развертываем в кластере
Пишем команду:

```
helm install postgresql oci://registry-1.docker.io/bitnamicharts/postgresql -f .\postgresql-values.yaml
```

В файле postgresql-values.yaml не забываем вписать свои данные для бд


## Secrets
Необходимо создать файл django-secrets.yaml и там разместить все чувствительные данные
Обязательно значение должно быть в кодировке base64
Пример файла с секретами:
```
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  SECRET_KEY: bXktc3VwZXItc2VjcmV0LWtleQ==
  DATABASE_URL: cG9zdGdyZXM6Ly90ZXN0X2s4czpPd090QmVwOUZydXRAMTcyLjIzLjMyLjc2OjU0MzIvdGVzdF9rOHM=
  DEBUG: VHJ1ZQ==
  ALLOWED_HOSTS: c3Rhci1idXJnZXIudGVzdA==

```
далее применяем эти секреты командой:
```
kubectl apply -f django-secrets.yaml
```

## Развертываем приложение по шагам
Установка Ingress:
```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```
Добавление для теста изменения в файл [hosts](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit)
```
192.168.59.100 star-burger.test
```

Запускаем файл deployment.yaml :
```
kubectl apply -f deployment.yaml
```

Запускаем файл service.yaml :
```
kubectl apply -f service.yaml
```

Запускаем файл ingress.yaml : 

```
kubectl apply -f ingress.yaml
```
Готово теперь приложение доступно по адресу http://star-burger.test

## Сборка образа и пуш в DockerHub

Для сборки образа с commit hash необходимо выполнить команды
Получение хэша коммита
```
export commit_hash=$(git rev-parse --short HEAD)

```
Сборка образа
```
docker build -t username/imagename: $commit_hash
```

Перед пушем необходимо залогиниться командой:
```
docker login
```

Пушим образ:
```
docker push username/imagename: $commit_hash
```


# Разворачиваем приложение на удаленном кластере
Необходимо получить доступ к удаленному кластеру, получить `namespace` которы необходимо указывать во всех манифестах
Пример инструкции к ресурсам кластера[Пример](https://sirius-env-registry.website.yandexcloud.net/edu-gloomy-ptolemy.html)

Можно вести работу в IDE с графическим интерфейсом например как [Lens Desktop](https://docs.k8slens.dev/getting-started/install-lens/)

## Подключение к БД
Для подключения к бд необходимо получить сертификат SSL
Добавить содержимое сертификата в кодировке base 64 в манифест cert-root.yaml и запустить командой:
```
kubectl apply -f ./cert-root.yaml
```
Для проверки подключения к БД запустите манифест psql.yaml

## Развертываем приложение по шагам
Необходимо создать файл django-secrets.yaml и там разместить все чувствительные данные
Обязательно значение должно быть в кодировке base64
Пример файла с секретами:
```
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
data:
  SECRET_KEY: bXktc3VwZXItc2VjcmV0LWtleQ==
  DATABASE_URL: cG9zdGdyZXM6Ly90ZXN0X2s4czpPd090QmVwOUZydXRAMTcyLjIzLjMyLjc2OjU0MzIvdGVzdF9rOHM=
  DEBUG: VHJ1ZQ==
  ALLOWED_HOSTS: c3Rhci1idXJnZXIudGVzdA==

```
далее применяем эти секреты командой:
```
kubectl apply -f django-secrets.yaml
```

Запускаем манифесты django-deployment.yaml и django-service.yaml:
```
kubectl apply -f django-deployment.yaml
kubectl apply -f django-service.yaml
```

Выполняем миграцию
```
kubectl apply -f django-migrate.yaml
```

Для создания суперпользователя необходимо войти в под, созданный манифестом django-deployment.yaml и там выполнить команду джанго manage.py createsuperuser

```
kubectl exec podname -it -- /bin/bash/
```

Добавляем автоматическую чистку сессий

```
kubectl apply -f django-clearsession.yaml
```


[Пример проекта](https://edu-gloomy-ptolemy.sirius-k8s.dvmn.org/)