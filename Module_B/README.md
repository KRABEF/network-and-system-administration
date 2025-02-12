# Модуль Б (Настройка технических и программных средств информационно-коммуникационных систем)

![Карта сети](img/netmap.png)

<table>
    <tr>
        <th>Название</th>
        <th>ОС</th>
    </tr>
    <tr>
        <td>R-HQ</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>R-DT</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>SW1-HQ</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>SW2-HQ</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>SW3-HQ</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>SRV1-HQ</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>FW-DT</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>SW-DT</td>
        <td>Alt linux Server 10</td>
    </tr>
    <tr>
        <td>SRV1-DT</td>
        <td>Alt linux Server 10</td>
    </tr>
</table>

### HQ офис

| Подсеть/VLAN | Префикс | Диапазон | Размер |
|:-|:-|:-|:-|
| 192.168.11.0 / vlan110 |/24 | 192.168.11.2-66 | 64 |
| 192.168.11.100 / vlan220 | /24 | 192.168.11.100-116 | 16 |
| 192.168.11.200 / vlan330 | /24 | 192.168.11.200-208 | 8 |

### DT офис

| Подсеть/VLAN | Префикс | Диапазон | Размер |
|:-|:-|:-|:-|
| 192.168.33.0 / vlan110 |/24 | 192.168.33.2-66 | 64 |
| 192.168.33.100 / vlan220 | /24 | 192.168.33.100-116 | 16 |
| 192.168.33.200 / vlan330 | /24 | 192.168.33.200-208 | 8 |

### Внеполосное управление виртуалками в Proxmox (на всякий)

<details>
  <summary>Нажми сюда))</summary>
  
Добавление serial порта в Гипервизоре:
```
qm set <VM ID> -serial0 socket
```
Хост `/etc/init/ttyS0.conf`:
```
# ttyS0 - getty
start on stopped rc RUNLEVEL=[12345]
stop on runlevel [!12345]
respawn
exec /sbin/getty -L 115200 ttyS0 vt102
```
Конфигурация `grub` `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX ='console=tty0 console=ttyS0,115200'
```
Update:
```
update-grub
```
Включение serial порта:
```
systemctl enable serial-getty@ttyS0.service
```
Перезагружаемся и заходим через `xterm.js`. Теперь доступны скроллинг, вставка, копирование и произвольный размер окна.

</details>

## Базовая настройка Alt Linux


<details>
  <summary>Установка имени ВМ</summary>

### Установка имени ВМ

(через рут)
```
hostnamectl set-hostname имя_пк
```

После этого выходим из системы или перезагружаемся.

Для проверки корректности установки имени ПК выполнить команду:
```bash
hostnamectl
# или
hostname
# или
cat /etc/hostname
```
</details>

<details>
<summary>Создание пользователя:</summary>

### Создание пользователя:

```bash
useradd -m -G wheel -s /bin/bash sshuser
```

Установка пароля:

```
passwd sshuser
P@ssw0rd
P@ssw0rd
```

Изменить права доступа к файлу `/etc/sudoers`:
```bash
chmod 600 /etc/sudoers
```
затем открыть файл в текстовом редакторе (vim, mcedit или nano) и раскомментировать строку: `WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL` для запуска утилиты sudo без дополнительной аутентификации. 

или выполнить команду:
```bash
echo "sshuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```
</details>


<details>
  <summary>Настройка SSH</summary>

### Настройка SSH

### Установка SSH сервера

```bash
apt-get install openssh-server -y

systemctl enable --now sshd
```

На сервере, к которому подключаемся
```bash
cd /home/sshuser
mkdir -p .ssh/
chmod 700 .ssh/
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
chown sshuser:sshuser .ssh/authorized_keys
```

Клиентская машина, с которой подключаемся
```bash
ssh-keygen -t rsa -b 2048 -f srv_ssh_key
mkdir ~/.ssh
mv srv_ssh_key* .ssh/
```

Конфиг для автоматического подключения (пример, т.к. айпи серверов могут отличаться) `.ssh/config`:
```
Host srv-hq
        HostName 192.168.11.100
        User sshuser
        IdentityFile .ssh/srv_ssh_key
        Port 22
Host srv-br
        HostName 192.168.33.100
        User sshuser
        IdentityFile .ssh/srv_ssh_key
        Port 22
```

```bash
chmod 600 .ssh/config
```
Копирование ключа на удаленный сервер:
```bash
ssh-copy-id -i .ssh/srv_ssh_key.pub sshuser@192.168.11.100
```
```bash
ssh-copy-id -i .ssh/srv_ssh_key.pub sshuser@192.168.33.100
```

На сервере `/etc/ssh/sshd_config`:
```
AllowUsers sshuser
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
AuthorizedKeysFile .ssh/authorized_keys
Port 22
```
перезапускаем службу:
```bash
systemctl restart sshd
```
Подключение:
```
ssh srv-hq
```
</details>

