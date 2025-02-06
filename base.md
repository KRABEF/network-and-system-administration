# Базовая настройка Alt Linux

## Установка Alt linux
### Создание пользователя во время установки:

имя пользователя - sshuser

пароль - P@ssw0rd

### или создание пользователя после установки:

```bash
useradd -m -G wheel -s /bin/bash sshuser
```

Установка пароля:

```
passwd sshuser
```

в файле `/etc/sudoers` раскомментировать строку: `WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL` для запуска утилиты sudo без дополнительноц аутентификации.

## Установка SSH сервера

```bash
apt-get install openssh-server -y

systemctl enable --now sshd
```

## Установка имени ПК
(через рут)
```
hostnamectl set-hostname имя_пк
```

Для проверки корректности установки имени ПК выполнить команду:
```bash
hostnamectl
# или
hostname
# или
cat /etc/hostname
```

