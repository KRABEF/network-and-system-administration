# Установка и настройка NTP

### 1. Установка NTP-клиента

```bash
apt-get install ntp -y
```

### 2. Запуск службы

```bash
systemctl start ntpd --now
```

### 3. Включение автозагрузки NTP

```bash
systemctl enable ntpd
```

```bash
systemctl status ntpd
```

### 4. Проверка синхронизации времени

Показывает список серверов времени, с которыми синхронизируется система, и их статус:

```bash
ntpq -p
```

### 5. Включение автоматической синхронизации времени

```bash
# Для синхронизации времени с внешними источниками:
apt-get install systemd-timesyncd
# Автосинхронизация времени через NTP:
timedatectl set-ntp true
```
