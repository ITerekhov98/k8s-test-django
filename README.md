# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).


## Развёрытвание с помощью Minikube

Перед началом работы убедитесь что у вас установлены следующие инструменты:
- Гипервизор. Если используете Windows, подойдёт [virtualbox](https://www.virtualbox.org/wiki/Downloads), либо [Hyper-V](https://msdn.microsoft.com/en-us/virtualization/hyperv_on_windows/quick_start/walkthrough_install). Для linux также можно использовать [virtualbox](https://www.virtualbox.org/wiki/Downloads), либо [KVM](https://www.linux-kvm.org/), который более прост и гибок в установке. Обратите внимание, что если вы используете WSL, вам понадобится провести дополнительные манипуляции. [Подробная инструкция](https://www.virtualizationhowto.com/2021/11/install-minikube-in-wsl-2-with-kubectl-and-helm/)

- Сам [Minicube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)

- Инструмент для управления кластерами Kubernetes: [Kubectl](https://kubernetes.io/docs/tasks/tools/)

Дальнейшие инструкции и команды будут актуальны для Ubuntu 20.04, в т.ч. установленной в WSL

### Запускаем кластер Minikube в нашей виртуальной среде:
```
minikube start
```
Как правило minikube по умолчанию выбирает нужный драйвер, но если возникнут проблемы, можно указать его явно через аргумент, например: `--vm-driver=virtualbox`.

Первый запуск займёт чуть больше времени чем последующие. Проверьте корректность утановки следующей командой:
```
kubectl cluster-info
```

### Настройка и запуск БД

- Скачайте репозиторий с кодом, перейдите в его директорию. Создайте файл `values.yaml`. Заполните его по шаблону, указав данные в следующем формате:
```
auth:
  enablePostgresUser: true
  postgresPassword: {your database password}
  username: {your username}
  password: {your user`s password}
  database: {your database`s name}
```

- Установите пакетный менеджер для Kubernetes - [Helm](https://helm.sh/).

Выполните следующие команды:
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgres -f values.yaml bitnami/postgresql
```
Helm установит и запустит postgresql, а также создаст пользователя по параметрам, указанным в `values.yaml`.


### Настройка и запуск Django
При развёртывании будет использоваться готовый image с Django. Если хотите использовать свой image, откройте файл kubernetes/django-service.yml` и измените значение image на адрес нужного на docker hub.


- Откройте `cubernetes/configmap.yml` и укажите нужные значения для Django. В DATABASE_URL укажите данные в формате 
`postgres://username:password@db_host:db_port/database_name`, где username, password, database_name - данные из файла `values.yaml`, db_host - значение, указанное в команде helm install, db_port - по умолчанию 5432

- Регистрируем конфиг и поднимаем Django:
```
kubectl apply -f kubernetes/configmap.yml
kubectl apply -f kubernetes/django-service.yml
```
Обратите внимание что в одном файле `kubernetes/django-service.yml` создаётся сразу Deployment и Service для нашего приложения.
- Создаём миграции:
```
kubectl apply -f kubernetes/migrates.yml
```
- Создаём kron службу для очистки устаревших сессий:
```
kubectl apply -f kubernetes/clearsessions.yml
```
- Запускаем балансировщик нагрузки - Ingress:

Добавьте выбранный хост(в примере используется star-burger.test) в `/etc/hosts`. Ip можно узнать с помощью команды `minikube ip`.

Выполните команду:

```
kubectl apply -f kubernetes/ingress.yml
```

### Проверка

Для просмотра запущенных сервисов выполните:
```
kubectl get svc
```
Чтобы получить ip и порт нужного сервиса можно также выполнить:
```
minikube service <your_service>
```





## Как запустить dev-версию с помощью docker-compose

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
