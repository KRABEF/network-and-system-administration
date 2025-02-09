# Конфигурация роутера на Alt linux

## Конфигурация VM
Нужно дать ВМ 2 сетевых адаптера:

1) Для внешней сети (enp0s3)
2) Для внутр. сети (enp0s8)
   
## Конфигурация DHCP-сервера
Установка DHCP-сервера
```bash
su-
apt-get update
apt-get install dhcp-server -y
```
### 1) На втором сетевом адаптере (enp0s8) настроить ip адрес:
В директории `/etc/net/ifaces/enp0s8` (если нет, то создать командой: `mkdir /etc/net/ifaces/enp0s8`)

Затем в директории создать файл с кофнигурацией ip адреса: `ipv4address`:
```
192.168.11.1/24
```

Создаем файл `options` с содержимым:
```
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPV4=YES
```

Перезапускаем сетевую службу:
```
systemctl restart network
```

### 2) настраиваем DHCP-сервер
В файле `/etc/dhcp/dhcpd.conf`
```
ddns-update-style none;

subnet 192.168.11.0 netmask 255.255.255.0 { #сеть и маска подсети
        option routers                  192.168.11.1; #адрес маршрутизатора
        option subnet-mask              255.255.255.0; #маска подсети

        option domain-name              "google.com"; #домен
        option domain-name-servers      8.8.8.8; #DNS-сервера для клиентов

        range dynamic-bootp 192.168.11.2 192.168.11.254; #диапазон DHCP-подсети

        #стандартное и максимальное время аренды (в секундах)
        #6 часов
        default-lease-time 21600;
        #12 часов
        max-lease-time 43200;
}
```

В файле `/etc/sysconfig/dhcpd` указать сетевой адаптер, на котором идет раздача ip адресов (enp0s8)
```
# The following variables are recognized:

DHCPDARGS=enp0s8

# Default value if chroot mode disabled.
#CHROOT="-j / -lf /var/lib/dhcp/dhcpd/state/dhcpd.leases"
```

Запускаем службу dhcp-сервера (из под рута):
```
systemctl enable dhcpd --now
```

Проверям работу:
```
systemctl status dhcpd
```

### 3) Настройка маршрутизации между двумя сетевыми картами

#### Включаем IP Forwarding
В файле `/etc/net/sysctl.conf` ставим значение строки `net.ipv4.ip_forward` в 1
```
net.ipv4.ip_forward = 1
```

Перезапускаем машину, проверяем IP Forwarding командой
```
sysctl net.ipv4.ip_forward
```
Правильным значением вывода этой команды должно быть:
```
net.ipv4.ip_forward = 1
```

#### Настраиваем NAT (маскарадинг)
Разрешаем трафик из локальной сети через интерфейс с интернетом:
```
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
iptables -A FORWARD -i enp0s3 -o enp0s8 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Сохранение конфигурации
```
iptables-save >> /etc/sysconfig/iptables
```

Запускаем службу iptables:
```
systemctl enable iptables.service --now
```
и проверяем службу iptables командой: `systemctl status iptables.service`

### Перезапуск виртуальной машины роутера:
```
reboot
```
