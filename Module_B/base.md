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

Изменить права доступа к файлу `/etc/sudoers`:
```bash
chmod 600 /etc/sudoers
```
затем открыть файл в текстовом редакторе и раскомментировать строку: `WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL` для запуска утилиты sudo без дополнительной аутентификации.

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

