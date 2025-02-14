# Часть 2 

## Управление доменом с помощью ADMC
1) Управление доменом с помощью ADMC осуществляться с ADMIN-HQ
2) Для подразделения CLI настройте политику изменения рабочего стола на картинку компании, а также запретите использование пользователям изменение сетевых настроек и изменение графических параметров рабочего стола.
3) Для подразделения ADMIN реализуйте подключение общей папки SAMBA с использованием доменных политик.

<details>
    <summary>Управление доменом с помощью ADMC</summary>

[Читай тут 1](https://www.altlinux.org/%D0%93%D1%80%D1%83%D0%BF%D0%BF%D0%BE%D0%B2%D1%8B%D0%B5_%D0%BF%D0%BE%D0%BB%D0%B8%D1%82%D0%B8%D0%BA%D0%B8/ADMC)

[Читай тут 2](https://docs.altlinux.org/ru-RU/archive/9.0/html/alt-workstation-e2k/ch35s02.html)

</details>

## Настройка межсетевого экрана
1. Сервера и Администраторы офиса DT должны иметь доступ ко всем устройствам
2. Клиенты офиса DT должны иметь доступ только к серверам
3. Разрешите ICMP-запросы администраторами офиса DT на внутренние интерфейсы межсетевого экрана

<details>
    <summary>Настройка межсетевого экрана</summary>

[Читай тут 1](https://www.altlinux.org/Etcnet_Firewall)

[Читай тут 2](https://www.altlinux.org/Firewall_start)

</details>


## Реализация бекапа общей папки на сервере SRV1-HQ с использованием systemctl
1) Бекап должен архивировать все данные в формат tar.gz и хранить в директории /var/bac/. 
	1. Архивация должна производиться благодаря юниту типа service с названием backup. 
	2. Сервис должен включатся автоматический при загрузке.
2) Время выполнение бекапа каждый день в 8 часов вечера. 
	1. Используйте юнит типа timer для выполнения. 
	2. Если устройство будет выключено, то архивация производится сразу после запуска.

<details>
    <summary>Реализация бекапа общей папки на сервере SRV1-HQ с использованием systemctl</summary>

### 1. Создание скрипта для бекапа
Создадим скрипт, который будет архивировать общую папку (например, `/srv/shared`) в формате tar.gz и сохранять архивы в директории `/var/bac/`.

Создаем файл:
```
nano /usr/local/bin/backup.sh
```
Добавляем в него следующий код:
```bash
#!/bin/bash
BACKUP_SRC="/srv/shared"
BACKUP_DEST="/var/bac"
DATE=$(date +%F_%H-%M-%S)
ARCHIVE_NAME="backup_$DATE.tar.gz"

# Создаем директорию для бекапов, если она не существует
mkdir -p $BACKUP_DEST

# Архивируем данные
tar -czvf $BACKUP_DEST/$ARCHIVE_NAME $BACKUP_SRC

# Удаляем архивы старше 7 дней (по желанию)
find $BACKUP_DEST -type f -name "backup_*.tar.gz" -mtime +7 -exec rm {} \;

```
Делаем скрипт исполняемым:
```
chmod +x /usr/local/bin/backup.sh
```

### 2. Создание Unit-файла для Service
Создаем юнит-файл для запуска архивации:
```
nano /etc/systemd/system/backup.service
```
Добавляем следующий код:
```
[Unit]
Description=Backup shared folder to /var/bac
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

[Install]
WantedBy=multi-user.target
```

### 3. Создание Timer для расписания
Создаем таймер, который будет запускать бекап каждый день в 8 вечера:
```
nano /etc/systemd/system/backup.timer
```
Добавляем следующий код:
```
[Unit]
Description=Daily backup at 8 PM

[Timer]
OnCalendar=*-*-* 20:00:00
Persistent=true

[Install]
WantedBy=timers.target
```
- `OnCalendar=*-*-* 20:00:00` — выполняется каждый день в 20:00.
- `Persistent=true` — если сервер был выключен, задача выполнится сразу после включения.

### 4. Активация и запуск
Перезагружаем демоны systemd:
```
systemctl daemon-reload
```
Активируем и запускаем юниты:
```
systemctl enable backup.service
systemctl enable backup.timer
systemctl start backup.timer
```

### 5. Проверка работы
Проверяем статус таймера:
```
systemctl status backup.timer
```
Можно вручную запустить бекап для проверки:
```
systemctl start backup.service
```
Проверяем наличие архивов в папке `/var/bac/`.

</details>


## Развертывание приложений в Docker на SRV2-DT
1) Создайте локальный Docker Registry.
2) Напишите Dockerfile для приложения web.
	1. В качестве базового образа используйте nginx:alpine
	2. Содержание index.html
```html    
<html>
	<body>
		<center><h1><b>WEB</b></h1></center>
	</body>
</html>
```
3) Соберите образ приложения web и загрузите его в ваш Registry.
		1. Используйте номер версии 1.0 для вашего приложения
		2. Образ должен быть доступен для скачивания и дальнейшего запуска на локальной машине
4) Разверните Docker контейнер используя образ из локального Registry.
	1. Имя контейнера web
    2. Контейнер должно работать на порту 80
    3. Обеспечьте запуск контейнера после перезагрузки компьютера


<details>
    <summary>Соберите образ приложения web и загрузите его в ваш Registry</summary>

### 1. Установка Docker
Если Docker еще не установлен, установим его:
```
apt-get update
apt-get install docker-engine -y
```
После успешной установки необходимо запустить сервис контейнеризации docker и добавить его в автозагрузку:
```
systemctl enable --now docker
```
Добавляем текущего пользователя в группу Docker (чтобы не использовать `sudo`):
```
usermod -aG docker $USER
```
Проверяем работоспособность:
```
docker version
docker info
```
### 2. Развертывание локального Docker Registry
#### 2.1. Запуск локального Registry
Запустим локальный Docker Registry на порту 5000:
```bash
docker run -d \
  --name registry \
  -p 5000:5000 \
  -v /var/lib/registry:/var/lib/registry \
  registry:2
```
Проверим, что контейнер работает:
```
docker ps
```

### 3. Создание Dockerfile для приложения web
Создадим директорию для проекта:
```
mkdir ~/web-app
cd ~/web-app
```
Создадим файл `Dockerfile`:
```
nano Dockerfile
```
Содержимое `Dockerfile`:
```
# Используем базовый образ nginx:alpine
FROM nginx:alpine

# Копируем файл index.html в директорию, откуда nginx будет обслуживать контент
COPY index.html /usr/share/nginx/html/index.html
```
Создаем файл `index.html`:
```
nano index.html
```
Содержимое ``index.html``:
```html
<html>
    <body>
        <center><h1><b>WEB</b></h1></center>
    </body>
</html>
```
### 4. Сборка Docker-образа
Собираем образ с версией 1.0:
```
docker build -t web:1.0 .
```
Проверяем наличие образа:
```
docker images
```
### 5. Загрузка образа в локальный Registry
#### 5.1. Тегируем образ
Docker ожидает, что имя реестра будет в начале имени образа:
```
docker tag web:1.0 localhost:5000/web:1.0
```
#### 5.2. Загрузка образа в локальный Registry
Загружаем образ:
```
docker push localhost:5000/web:1.0
```
Проверяем, что образ загружен:
```
curl http://localhost:5000/v2/_catalog
```
### 6. Развертывание контейнера из локального Registry
#### 6.1. Запуск контейнера
Запускаем контейнер с именем `web` на порту 80:
```
docker run -d \
  --name web \
  -p 80:80 \
  --restart unless-stopped \
  localhost:5000/web:1.0
```
- `-d` — запускает контейнер в фоне.
- `-p 80:80` — маппинг порта 80 контейнера на порт 80 хоста.
- `--restart unless-stopped` — контейнер будет автоматически запускаться после перезагрузки.
#### 6.2. Проверка работы контейнера
Проверяем состояние контейнера:
```
docker ps
```
Заходим в браузер по адресу:
```
http://<IP_сервера>
```
или
```
http://localhost
```
или используем утилиту `curl`:
```
curl http://localhost/
```
и получаем ответ:
```
<html>
    <body>
        <center><h1><b>WEB</b></h1></center>
    </body>
</html>
```
```
curl -I http://localhost/
```
результат
```
HTTP/1.1 200 OK
Server: nginx/1.27.4
Date: Fri, 14 Feb 2025 13:09:32 GMT
Content-Type: text/html
Content-Length: 83
Last-Modified: Fri, 14 Feb 2025 12:44:35 GMT
Connection: keep-alive
ETag: "67af3ab3-53"
Accept-Ranges: bytes
```
### Фикс проблемы
```
docker: Error response from daemon: driver failed programming external connectivity on endpoint web (80faa3c2e3085af1bb482ea9d07c9d03eee5ed3125c882aab6ce8c8d20fa1497): failed to bind port 0.0.0.0:80/tcp: Error starting userland proxy: listen tcp4 0.0.0.0:80: bind: address already in use.
```

#### 1. Проверка, что занимает порт 80
Чтобы узнать, какой процесс использует порт 80, выполните:
```
ss -tuln | grep :80
```
```
netstat -tuln | grep :80
```
Если нужно узнать PID процесса, который занимает порт, выполните:
```
lsof -i :80
```
Если используете Alt linux server, то скорее всего, вы увидите
```
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
httpd2  2794    root    4u  IPv6  16031      0t0  TCP *:http (LISTEN)
httpd2  2881 apache2    4u  IPv6  16031      0t0  TCP *:http (LISTEN)
httpd2  2882 apache2    4u  IPv6  16031      0t0  TCP *:http (LISTEN)
```
Веб сервер `apache2`, который нужно остановить и выключить из автозагрузки:
```
systemctl stop httpd2.service
systemctl disable httpd2.service
```

#### 2. Проверить контейнеры Docker
Возможно, другой контейнер уже использует порт 80. Проверим запущенные контейнеры:
```
docker ps
```
Если есть контейнер, использующий порт 80, остановите его:
```
docker stop <container_id>
```
Или удалите его, если он больше не нужен:
```
docker rm <container_id>
```
После освобождения порта попробуйте снова запустить контейнер:
```
docker run -d \
  --name web \
  -p 80:80 \
  --restart unless-stopped \
  localhost:5000/web:1.0
```

#### 3. Альтернативное решение: Использовать другой порт
Если освобождать порт 80 не вариант, можно запустить контейнер на другом порту, например, 8080:
```
docker run -d \
  --name web \
  -p 8080:80 \
  --restart unless-stopped \
  localhost:5000/web:1.0
```

</details>