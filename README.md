# Домашнее задание к занятию 5. «Практическое применение Docker»

## Задача 0

Проверил свой сервер, т.к. спредыдущей домашней работы уже были установлены пакеты Docker и Docker compose просто проверил версию командой 
```
root@compute-vm-2-2-20-ssd-1783340222914:~/homework/shvirtd-example-python# docker compose version
Docker Compose version v5.3.1
```
## Задача 1

1. Сделал форк [Репозиторий](https://github.com/Folau1/shvirtd-example-python)

2. Создаем файлы Dockerfile.python на основе Dockerfile в репозитории.

Для теста сделал git clone на сервер.
Добавил все файлы:
```
ls -a
.              .env        Dockerfile.python  haproxy  proxy.yaml
..             .git        LICENSE            main.py  requirements.txt
.dockerignore  Dockerfile  README.md          nginx    schema.pdf
```
Собрал образ и запустил
```
docker build -t projectx -f Dockerfile.python .
docker run projectx
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:5000 (Press CTRL+C to quit)
```

Для multistage сборки нужно переделать немного рабочий Dockerfile.python
```
FROM python:3.12-slim AS builder

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim AS runtime

WORKDIR /app

COPY --from=builder /install /usr/local

COPY . .

# Запускаем приложение с помощью uvicorn, делая его доступным по сети
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5000"]
```

Пересобираем и проверяем:
```
docker build -t projectx -f Dockerfile.python .
docker run projectx
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:5000 (Press CTRL+C to quit)
```
3. (Необязательно задание)
Python код можно запускать и не через докер. Например через виртуальное пространство venv.
Сначала добавим все лишние файлы в gitignore
```
.git
.env
venv
__pycache__
*.pyc
```

Проверяем версию, создаем вирт пространство и входим в него:
```
python3 --version
Python 3.12.3
python3 -m venv venv
source venv/bin/activate
(venv) root@compute-vm-2-2-20-ssd-1783340222914:~/homework/shvirtd-example-python# 
```
С помощью команды python -m pip install -r requirements.txt мы устанавливаем зависимости внутри вирт пространства.
venv создаёт отдельный каталог с интерпретатором и установленными библиотеками. Активация просто ставит его bin в начало PATH. Это всё написано [Тут](https://docs.python.org/3/library/venv.html)
Сам MYSQL можно запустить через RUN по заданию.
На официальной[странице](https://hub.docker.com/_/mysql) можно подробно посмотреть как запускать через run MYSQL.
Примерно по такому же принципу делает так:
```
source .env
docker run -d --name mysql -p 127.0.0.1:3306:3306 -e MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" -e MYSQL_DATABASE="$MYSQL_DATABASE" -e MYSQL_USER="$MYSQL_USER" -e MYSQL_PASSWORD="$MYSQL_PASSWORD" mysql:8.0
```

И теперь на порту 5000 запускаем наше python приложение через unicor 
```
uvicorn main:app --host 0.0.0.0 --port 5000 --reload
INFO:     Will watch for changes in these directories: ['/root/homework/shvirtd-example-python']
INFO:     Uvicorn running on http://0.0.0.0:5000 (Press CTRL+C to quit)
INFO:     Started reloader process [81119] using WatchFiles
INFO:     Started server process [81121]
INFO:     Waiting for application startup.
Приложение запускается...
Соединение с БД установлено и таблица 'requests' готова к работе.
INFO:     Application startup complete.
```
## Задача 2 (*)

Устанавливаем Yandex Cloud CLI по официальной [инструкции](https://yandex.cloud/ru/docs/cli/operations/install-cli).

Создаём реестр:

```console
yc container registry create --name test
done (1s)
id: crpbd1dq093c89j0vma9
folder_id: b1g40q4ai8pdtrbga82v
name: test
status: ACTIVE
created_at: "2026-07-22T07:26:57.768Z"
```

Настраиваем Docker на авторизацию в Yandex Cloud:

```console
yc container registry configure-docker
docker configured to use yc --profile "packer-sa" for authenticating "cr.yandex" container registries
Credential helper is configured in '/root/.docker/config.json'
```

Загружаем образ и проверяем его наличие в реестре:

```console
docker push cr.yandex/crpbd1dq093c89j0vma9/projectx:latest
yc container image list --repository-name=crpbd1dq093c89j0vma9/projectx

+----------------------+---------------------+-------------------------------+-------------------------------------------------------------------------+-----------------+
|          ID          |       CREATED       |             NAME              |                                  TAGS                                   | COMPRESSED SIZE |
+----------------------+---------------------+-------------------------------+-------------------------------------------------------------------------+-----------------+
| crp1ettknogdlkdo3ueg | 2026-07-22 07:31:58 | crpbd1dq093c89j0vma9/projectx | latest                                                                  | 84.5 MB         |
| crpei3sufldp160lfjvv | 2026-07-22 07:31:58 | crpbd1dq093c89j0vma9/projectx | sha256:d594bcc2106b14da1c7bd81f00542199698f1a5273976090f99ed9173923ee1a | 84.5 MB         |
| crptcfcffg39m3sl39ra | 2026-07-22 07:31:57 | crpbd1dq093c89j0vma9/projectx | sha256:95d43b6fbb9cd0f8b83e2a48ddbb5a4cb71f8e27f2c04071a6d9241902c44900 | 1.7 KB          |
+----------------------+---------------------+-------------------------------+-------------------------------------------------------------------------+-----------------+
```

## Задача 3 
1. Изучив proxy.yaml можно увидеть, что это compos файл для haproxy и nginx.
Если быть точнее, то reverse-proxy использует "haproxy:2.4" принимает запрос от приложения backend и принимает запросы на порту 8080.
ingress-proxy использует "nginx:latest" и принимает внешние запросы на порту 8090.
Получается, мы используем двойные прокси, внутренние и внешние для проксирования запросов. HAProxy внутренний балансировщик, Nginx извне принимает запрос.
2. Создал файл compose.yaml и подключил в самом начале proxy.yaml
3. Полный compose файл:

```
include:
  - proxy.yaml

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.python #Говорим docker чтобы использовал наш файл
    container_name: web
    restart: always
    environment:   #задаём данные как делали мы на вебинаре
      DB_HOST: db
      DB_USER: ${MYSQL_USER}
      DB_PASSWORD: ${MYSQL_PASSWORD}
      DB_NAME: ${MYSQL_DATABASE}
    depends_on:
      db:
        condition: service_healthy
    networks:       #Подключаем web к сети
      backend:
        ipv4_address: 172.20.0.5    #Это стандартный айпишник который используется в>

  db:      #Наша база данных! mysql 8
    image: mysql:8
    container_name: db
    networks:
      backend:
        ipv4_address: 172.20.0.10
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    env_file:
      - .env
    healthcheck:     #тут проверяем готовность Mysql
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -uroot -p$${MYSQL_ROOT_PASSWO>
      interval: 10s
      timeout: 5s
      retries: 20
      start_period: 20s
    volumes:
      - mysql_data:/var/lib/mysql

volumes:    #Том для mysql
  mysql_data:
```

Далее билдим и запускаем проект командами:
```
docker compose build
docker compose up -d
```
4. Подключаемся к нашей MYSQL Базе через .env файл внутри контейнера:
```
source .env
docker exec -ti db mysql -uroot -p"$MYSQL_ROOT_PASSWORD"
```
Результат запросов ниже на скриншоте:
<img width="744" height="607" alt="image" src="https://github.com/user-attachments/assets/f32c9d2c-3977-4182-8c45-6e49ae362b84" />

## Задача 4
Задание с 1-2 пропускаем, т.к. я делал всё внутри ВМ Яндекс Клауда.
3. bash скрипт для автозапуска:
```
#!/usr/bin/env bash

set -e
#Значения:
REPO_URL="https://github.com/Folau1/shvirtd-example-python.git"   #наш репозиторий
APP_DIR="/opt/shvirtd-example-python"

#Логика:
git clone "$REPO_URL" "$APP_DIR"
cd "$APP_DIR"

docker compose up -d --build
docker compose ps
```

Делаем скрипт исполняемым:
```
chmod +x deploy.sh
```
Запускаем:
```
./deploy.sh
```
5. Remote ssh контекст к моему серверу. делал со своего ПК.
Вот вывод:
```
S C:\Users\Александр\Documents\Netology_work> docker ps -a
CONTAINER ID   IMAGE                         COMMAND                  CREATED          STATUS                      PORTS                                     NAMES
bd2c61eb2d8d   shvirtd-example-python-web    "uvicorn main:app --…"   46 minutes ago   Up 46 minutes                                                         web
665b7d4c188d   nginx:latest                  "/docker-entrypoint.…"   46 minutes ago   Up 46 minutes                                                         shvirtd-example-python-ingress-proxy-1
bde3e0cd56a6   haproxy:2.4                   "docker-entrypoint.s…"   46 minutes ago   Up 46 minutes               127.0.0.1:8080->8080/tcp                  shvirtd-example-python-reverse-proxy-1
a38fcb967fa5   mysql:8                       "docker-entrypoint.s…"   46 minutes ago   Up 46 minutes (healthy)     3306/tcp, 33060/tcp                       db
46a106f0e760   mysql:8.0                     "docker-entrypoint.s…"   5 hours ago      Exited (0) 2 hours ago                                                mysql-venv
3d2ca3170e70   projectx                      "uvicorn main:app --…"   5 hours ago      Exited (0) 5 hours ago                                                charming_snyder
e4591c959b2d   b297b542fd73                  "uvicorn main:app --…"   5 hours ago      Exited (0) 5 hours ago                                                gifted_sutherland
69eb1b261f3d   b297b542fd73                  "uvicorn main:app --…"   5 hours ago      Exited (0) 5 hours ago                                                musing_rubin
d1c6b9db789a   b297b542fd73                  "uvicorn main:app --…"   5 hours ago      Exited (0) 5 hours ago                                                magical_booth
b6eedce3a49d   b297b542fd73                  "uvicorn main:app --…"   5 hours ago      Exited (0) 5 hours ago                                                recursing_chaum
8c581dc0ed06   127.0.0.1:5000/custom-nginx   "/docker-entrypoint.…"   8 days ago       Exited (255) 31 hours ago   0.0.0.0:9090->80/tcp, [::]:9090->80/tcp   dep-nginx-1
ee0bd091da4e   debian:12                     "tail -f /dev/null"      8 days ago       Exited (255) 31 hours ago                                             debian-t4
eaae7bc2923b   centos:7                      "tail -f /dev/null"      8 days ago       Exited (255) 31 hours ago                                             centos-t4
5b24b3ef03f8   hello-world                   "/hello"                 2 weeks ago      Exited (0) 2 weeks ago                                                blissful_pike
PS C:\Users\Александр\Documents\Netology_work> docker context ls
NAME          DESCRIPTION                               DOCKER ENDPOINT                  ERROR
default       Current DOCKER_HOST based configuration   npipe:////./pipe/docker_engine   
yandex-vm *   Яндекс Клауд                              ssh://yandex-vm                  
PS C:\Users\Александр\Documents\Netology_work> 
```
<img width="1082" height="824" alt="image" src="https://github.com/user-attachments/assets/6fb3fea5-a2cf-4c11-a5f4-ff7ba57b61be" />

5. Скриншоты запросов:
<img width="1079" height="914" alt="image" src="https://github.com/user-attachments/assets/ebb1be45-1c5e-4662-a040-dd94d23dcfba" />
Последние:
<img width="1081" height="925" alt="image" src="https://github.com/user-attachments/assets/5019821b-1006-4e74-b6cd-78547910c680" />


