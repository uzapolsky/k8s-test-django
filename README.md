# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки используйте переменные окружения. Список доступных переменных можно найти внутри файла `docker-compose.yml`.

## Как развернуть сайт с помощью Kubernetes

Установить [Kubernetes](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/).

Установить [Minicube](https://minikube.sigs.k8s.io/docs/start/).

Установить [VirtualBox](https://www.virtualbox.org/).

Установить [Helm](https://helm.sh/).

Добавить ноду minikube:

```shell-session
$ minikube start --driver=virtualbox
```

Собрать образ сайта для создания сервиса:

```shell-session
$ minikube image build -t django_app backend_main_django
```

Поднять базу данных:

```shell-session
$ helm install django-psql bitnami/postgresql
```

Заполнить файл `django-config-example.yml` своими переменными окружения, переименовать его в `django-config.yml` и применить `configMap` командой в новом терминале:

```shell-session
$ kubectl apply -f django-config.yml
```

Запустить `ingress `:

```shell-session
$ minikube addons enable ingress
$ kubectl apply -f django-ingress.yml
```

Запустить `cronjob` для автоматического удаления сессий:

```
$ kubectl apply -f django-cronjob.yml
```

Запустить сервис:

```shell-session
$ kubectl apply -f django-service.yml
```

Сделать сервис доступным по адресу `example.test`:

```shell-session
echo "$(minikube ip) example.test" | sudo tee -a /etc/hosts
```

Для запуска миграций необходимо выполнить команду:

```shell-session
kubectl apply -f django-migrate-job.yml
```

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).