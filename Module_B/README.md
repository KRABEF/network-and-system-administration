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
| 192.168.11.0 / vlan110 |/24 | 192.168.11.1-65 | 64 |
| 192.168.11.66 / vlan220 | /24 | 192.168.11.66-82 | 16 |
| 192.168.11.83 / vlan330 | /24 | 192.168.11.83-91 | 8 |

### DT офис

| Подсеть/VLAN | Префикс | Диапазон | Размер |
|:-|:-|:-|:-|
| 192.168.33.0 / vlan110 |/24 | 192.168.33.1-65 | 64 |
| 192.168.33.66 / vlan220 | /24 | 192.168.33.66-82 | 16 |
| 192.168.33.83 / vlan330 | /24 | 192.168.33.83-91 | 8 |

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


## Настройка роутера на базе Alt Linux

<details>
  <summary>Настройка роутера R-HQ</summary>

### Настройка роутера R-HQ

#### Конфигурация VM
Нужно дать ВМ 2 сетевых адаптера:

1) Для внешней сети (enp0s3)
2) Для внутр. сети (enp0s8)

<b>Внимание: название сетевых адаптеров в вашем случае может отличаться!!</b>

Первым делом нужно настроить IP адрес от провайдера, в нашем случае, это `172.16.5.15` в сети `172.16.5.0/28`, где 16 доступных адресов для адресации. Таким образом, нам нужен старший адрес, которым является `15`.

```
vim /etc/net/ifaces/enp0s3/options
```
```
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPV4=YES
```
```bash
echo "172.16.5.15/28" > /etc/net/ifaces/enp0s3/ipv4address
```
```bash
echo "default via 172.16.5.1" > /etc/net/ifaces/enp0s3/ipv4route
```


#### Конфигурация DHCP-сервера
Установка DHCP-сервера
```bash
su-
apt-get update
apt-get install dhcp-server -y
```

#### 1) Настроить второй сетевой адаптер, в нашем случае, это enp0s8:
1) `vim /etc/net/ifaces/enp0s8/options`
    ```
    TYPE=eth
    DISABLED=no
    NM_CONTROLLED=no
    BOOTPROTO=static
    CONFIG_IPV4=YES
    ```
2) Настройка модуля ядра для работы с vlan
    ```
    modprobe 8021q
    echo "8021q" >> /etc/modules
    ```

    Проверяем:
    ```
    lsmod | grep 8021q
    ```

#### 2) Настройка VLAN
Создаем виртуальные интерфейсы для VLAN на `enp0s8`

1) <b>VLAN 110 (Клиенты)</b>
    ```
    mkdir -p /etc/net/ifaces/enp0s8.110
    echo "192.168.11.1/26" > /etc/net/ifaces/enp0s8.110/ipv4address
    ```
    Создаем файл `options`:
    ```
    TYPE=vlan
    HOST=enp0s8
    BOOTPROTO=static
    # CONFIG_IPV4=yes
    VID=110
    ```
2) <b>VLAN 220 (Сервера)</b>
    ```
    mkdir -p /etc/net/ifaces/enp0s8.220
    echo "192.168.11.65/28" > /etc/net/ifaces/enp0s8.220/ipv4address
    ```
    Создаем файл `options`:
    ```
    TYPE=vlan
    HOST=enp0s8
    BOOTPROTO=static
    # CONFIG_IPV4=yes
    VID=220
    ```
3) <b>VLAN 330 (Администраторы)</b>
    ```
    mkdir -p /etc/net/ifaces/enp0s8.330
    echo "192.168.11.81/29" > /etc/net/ifaces/enp0s8.330/ipv4address
    ```
    Создаем файл `options`:
    ```
    TYPE=vlan
    HOST=enp0s8
    BOOTPROTO=static
    # CONFIG_IPV4=yes
    VID=330
    ```

Перезапускаем сетевую службу:
```
systemctl restart network
```

#### 3) настраиваем DHCP-сервер
В файле `/etc/dhcp/dhcpd.conf`
```
ddns-update-style none;

# VLAN 110 (Клиенты)
subnet 192.168.11.0 netmask 255.255.255.192 {
        option routers                  192.168.11.1;
        option subnet-mask              255.255.255.192;
        option domain-name              "au.team";
        option domain-name-servers      8.8.8.8;
        range dynamic-bootp 192.168.11.2 192.168.11.62;
}

# VLAN 220 (Сервера)
subnet 192.168.11.64 netmask 255.255.255.240 {
        option routers                  192.168.11.65;
        option subnet-mask              255.255.255.240;
        option domain-name              "au.team";
        option domain-name-servers      8.8.8.8;
        range dynamic-bootp 192.168.11.66 192.168.11.78;
}

# VLAN 330 (Администраторы)
subnet 192.168.11.80 netmask 255.255.255.248 {
        option routers                  192.168.11.81;
        option subnet-mask              255.255.255.248;
        option domain-name              "au.team";
        option domain-name-servers      8.8.8.8;
        range dynamic-bootp 192.168.11.82 192.168.11.86;
}

```

В файле `/etc/sysconfig/dhcpd` указать сетевой адаптер, на котором идет раздача ip адресов (enp0s8)
```
DHCPDARGS="enp0s8.110 enp0s8.220 enp0s8.330"
```

Запускаем службу dhcp-сервера (из под рута):
```
systemctl enable dhcpd --now
```

Проверям работу:
```
systemctl status dhcpd
```

#### 3) Настройка маршрутизации между двумя сетевыми картами

#### Включаем IP Forwarding
В файле `/etc/net/sysctl.conf` ставим значение строки `net.ipv4.ip_forward` в 1
```
net.ipv4.ip_forward = 1
```

Применяем настройки вручную:
```
sysctl -p /etc/net/sysctl.conf
```

или перезапускаем машину, проверяем IP Forwarding командой
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
# Маскарадинг для доступа в интернет
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE

# Маршрутизация между VLAN
iptables -A FORWARD -i enp0s8.110 -o enp0s8.220 -j ACCEPT
iptables -A FORWARD -i enp0s8.220 -o enp0s8.110 -j ACCEPT

iptables -A FORWARD -i enp0s8.110 -o enp0s8.330 -j ACCEPT
iptables -A FORWARD -i enp0s8.330 -o enp0s8.110 -j ACCEPT

iptables -A FORWARD -i enp0s8.220 -o enp0s8.330 -j ACCEPT
iptables -A FORWARD -i enp0s8.330 -o enp0s8.220 -j ACCEPT

# Доступ в интернет
iptables -A FORWARD -i enp0s8.110 -o enp0s3 -j ACCEPT
iptables -A FORWARD -i enp0s8.220 -o enp0s3 -j ACCEPT
iptables -A FORWARD -i enp0s8.330 -o enp0s3 -j ACCEPT

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

#### Перезапуск виртуальной машины роутера:
```
reboot
```

</details>


<details>
  <summary>Настройка роутера R-DT</summary>

### Настройка роутера R-DT

Настройка аналогична настройке роутера R-HQ за исключением конфигурации входящей и выходящей сетей. 

<b>Внимание: название сетевых адаптеров в вашем случае может отличаться!!</b>

</details>

### Настройка клиентов для работы в сетях VLAN
<details>
  <summary>Настройка клиентов для работы в сетях VLAN</summary>

1) Включение поддержки VLAN
Загрузите модуль:
```
modprobe 8021q
```
Чтобы модуль загружался при старте системы:
```
echo "8021q" | tee -a /etc/modules
```

2) Настройка интерфейсов в etcnet

    Конфигурационные файлы находятся в /etc/net/ifaces/. 
    Пусть основной интерфейс — enp0s3. Настроим VLAN 110, 220, 330.

    1) **Основной интерфейс (enp0s3)**  
    Файл: `/etc/net/ifaces/enp0s3/options`
        ```
        TYPE=eth
        DISABLED=no
        NM_CONTROLLED=no
        BOOTPROTO=none
        ```
    2) **VLAN 110 (Клиенты)**   
    Файл: `/etc/net/ifaces/enp0s3.110/options`
        ```
        TYPE=vlan
        BOOTPROTO=dhcp
        VID=110
        CONFIG_IPV4=yes
        HOST=enp0s3
        ONBOOT=yes
        ```
    3) **VLAN 220 (Сервера)**   
    Файл: `/etc/net/ifaces/enp0s3.220/options`
        ```
        TYPE=vlan
        BOOTPROTO=dhcp
        VID=220
        CONFIG_IPV4=yes
        HOST=enp0s3
        ONBOOT=yes
        ```
    4) **VLAN 330 (Администраторы)**   
    Файл: `/etc/net/ifaces/enp0s3.330/options`
        ```
        TYPE=vlan
        BOOTPROTO=dhcp
        VID=330
        CONFIG_IPV4=yes
        HOST=enp0s3
        ONBOOT=yes
        ```
3) **Применение изменений**
Перезапустите сетевую службу:
```bash
sudo systemctl restart network
```
4) Проверка работы
Проверка полученных IP-адресов:
```
ip -c a
```
```
ip route
```
```
ping 192.168.11.1
```
</details>


## Настройка коммутаторов

<details>
  <summary>Настройка SW-HQ</summary>

### Настройка SW-HQ

1) В качестве коммутаторов используются: SW1-HQ, SW2-HQ, SW3-HQ, SW-DT
2) Для каждого офиса устройства должны находиться в соответствующих VLAN:
	- Клиенты - vlan110,
	- Сервера – в vlan220,
	- Администраторы – в vlan330.

3) Создайте management интерфейсы на коммутаторах

#### Конфигурация

актуально для SW1-HQ, SW2-HQ и SW3-HQ

<b>P.s. Внимание! в интерфейсах ens... не должно быть прочих файлов, кроме options, иначе порты не привяжутся</b>

P.s. Название сетевых адаптеров в зависимости от используемого гипервизора может отличаться!

Временное назначение ip-адреса (смотрящего в сторону r-hq):
```
ip link add link ens18 name ens18.330 type vlan id 330
ip link set dev ens18.330 up
ip addr add 192.168.11.200 dev ens18.330
ip route add 0.0.0.0/0 via 192.168.11.1
echo nameserver 8.8.8.8 > /etc/resolv.conf
```

Обновление пакетов и установка `openvswitch`:
```bash
apt-get update && apt-get install -y openvswitch
```

Включение `ovs` в автозагрузку:
```
systemctl enable --now openvswitch
```

проверка службы `ovs`:
```
systemctl status openvswitch
```

1. ens18 - R-HQ
2. ens19 - _ - vlan220
3. ens20 - srv-hq - vlan110 
4. ens21 - __ - vlan220

Создаем каталоги для ens19,ens20,ens21:
```
mkdir /etc/net/ifaces/ens{19,20,21}
```

Для моста:
```
mkdir /etc/net/ifaces/ovs0
```
Management интерфейс:
```
mkdir /etc/net/ifaces/mgmt
```
Не удалять настройки интерфейсов openvswitch:
```
sed -i "s/OVS_REMOVE=yes/OVS_REMOVE=no/g" /etc/net/ifaces/default/options
```
Конфигурируем мост `ovs0`:
```
TYPE=ovsbr
HOST='ens18 ens19 ens20 ens21'
```
>- TYPE - тип интерфейса, bridge;
>- HOST - добавляемые интерфейсы в bridge.

Конфигурация `mgmt`:
`/etc/net/ifaces/mgmt/options`
```
TYPE=ovsport
BOOTPROTO=static
CONFIG_IPV4=yes
BRIDGE=ovs0
VID=330
```
> TYPE - тип интерфейса (internal);  
> BOOTPROTO - статически;  
> CONFIG_IPV4 - использовать ipv4;  
> BRIDGE - определяет к какому мосту необходимо добавить данный интерфейс;  
> VID - определяет принадлежность интерфейса к VLAN.  

Поднимаем сетевые интерфейсы:
```
echo -e "TYPE=ovsport\nCONFIG_IPV4=no\nONBOOT=yes\nBRIDGE=ovsbr0" | sudo tee /etc/net/ifaces/ens18/options
```

```
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens19/options
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens20/options
cp /etc/net/ifaces/ens18/options /etc/net/ifaces/ens21/options
```

Назначаем Ip, default gateway на mgmt:
```
echo 192.168.11.201/24 > /etc/net/ifaces/mgmt/ipv4address
```
```
echo default via 10.0.10.200 > /etc/net/ifaces/mgmt/ipv4route
```
Перезапуск network:
```
systemctl restart network
```
Проверка:
```
ip -c --br a
ovs-vsctl show
```

ens18 - R-HQ делаем trunk и пропускаем VLANs:
```
ovs-vsctl set port ens18 trunk=110,220,330
```
ens19 - tag=220
```
ovs-vsctl set port ens19 tag=220
```
ens20 - tag=110:
```
ovs-vsctl set port ens20 tag=110
```
ens21 - tag=220
```
ovs-vsctl set port ens21 tag=220
```
Включаем инкапсулирование пакетов по 802.1q:
```
modprobe 8021q
```
Проверка:
```bash
lsmod | grep 8021q
```
Результат:
```
8021q           40960   0
garp            16384   1   8021q
mrp             20480   1   8021q
```

</details>

## Настройка DNS для SRV-HQ и SRV-BR

i.	Реализуйте основной DNS сервер компании на SRV-HQ  
&ensp; a.	Для всех устройств обоих офисов необходимо создать записи A и PTR.  
&ensp; b.	Для всех сервисов предприятия необходимо создать записи CNAME.  
&ensp; c.	Создайте запись test таким образом, чтобы при разрешении имени из левого офиса имя разрешалось в адрес SRV-HQ, а из правого – в адрес SRV-BR.  
&ensp; d.	Сконфигурируйте SRV-BR, как резервный DNS сервер. Загрузка записей с SRV-HQ должна быть разрешена только для SRV-BR.  
&ensp; e.	Клиенты предприятия должны быть настроены на использование внутренних DNS серверов  

[Взято тут](https://github.com/abdurrah1m/Professionals_2024)

**Внимание: IP адреса отличаются!! См. выше.**

<details>
  <summary>Настройка DNS для SRV-HQ и SRV-BR</summary>

### SRV-HQ

Установка bind и bind-utils:
```
apt-get update && apt-get install -y bind bind-utils
```
Конфиг:
```
nano /etc/bind/options.conf
```
```
listen-on { any; };
allow-query { any; };
allow-transfer { 10.0.20.2; }; 
```

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/12ad33f9-2df1-47e7-ad7a-1788b2276c88)

Включаем resolv:
```
nano /etc/net/ifaces/ens18/resolv.conf
```
```
systemctl restart network
```
Автозагрузка bind:
```
systemctl enable --now bind
```
Создаем прямую и обратные зоны:
```
nano /etc/bind/local.conf
```

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/74bbb72d-1577-413d-969c-bff4497d0b9c)

Копируем дефолты:
```
cp /etc/bind/zone/{localhost,company.db}
```
```
cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/10.0.10.in-addr.arpa.db
```
```
cp /etc/bind/zone/127.in-addr.arpa /etc/bind/zone/20.0.10.in-addr.arpa.db
```
Назначаем права:
```
chown root:named /etc/bind/zone/company.db
```
```
chown root:named /etc/bind/zone/10.0.10.in-addr.arpa.db
```
```
chown root:named /etc/bind/zone/20.0.10.in-addr.arpa.db
```
Настраиваем зону прямого просмотра `/etc/bind/zone/company.prof`:

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/9d3bddf9-e8db-4cfc-8971-b3a6ff0f38ac)

Настраиваем зону обратного просмотра `/etc/bind/zone/10.0.10.in-addr.arpa.db`:

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/3fee1532-458b-4e10-967f-b858a4a43b63)

Настраиваем зону обратного просмотра `/etc/bind/zone/20.0.10.in-addr.arpa.db`:

![изображение](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/c9f18689-b324-4125-836d-91bdb23b1075)

Проверка зон:
```
named-checkconf -z
```
![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/d2de05e9-3830-4846-86a1-2974bb64dbf5)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/e1b6fb80-63df-4cf5-8dd6-2f3d46a1dc3b)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/21d53c0b-27d9-4af9-bbe3-b64035b0ffad)

### SRV-BR

Конфиг
```
vim /etc/bind/options.conf
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/e90d49ce-6735-4fdb-b44a-0b1c62b8305a)

Добавляем зоны

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/ea22291d-0b41-4044-a271-1fbb32f26185)

Резолв `/etc/net/ifaces/ens18/resolv.conf`:
```
search company.prof
nameserver 10.0.10.2
nameserver 10.0.20.2
```
Перезапуск адаптера:
```
systemctl restart network
```
Автозагрузка:
```
systemctl enable --now bind
```
SLAVE:
```
control bind-slave enabled
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/dc174c7e-e960-42c6-8b9d-7522df00989a)

Разрешение имени хоста test

### SRV-HQ

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/a73800a1-b7cf-48e3-8160-49b3b14a0060)

Копируем дефолт для зоны:
```
cp /etc/bind/zone/{localdomain,test.company.db}
```

Задаём права, владельца:
```
chown root:named /etc/bind/zone/test.company.db
```
Настраиваем зону:
```
vim /etc/bind/zone/test.company.db
```

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/ae1dcd6e-d5b9-4980-8105-d14b74905083)

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/c8735610-56d5-419b-bdc8-3efc48a969b0)

Перезапускаем:
```
systemctl restart bind
```

### SRV-BR
Добавляем зону `/etc/bind/local.conf`:

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/2d279b81-8bfd-453b-8efa-dc3d87476c40)

Задаём права, владельца:
```
chown root:named /etc/bind/zone/test.company.db
```

Редактируем зону `/etc/bind/zone/test.company.db`:

![image](https://github.com/abdurrah1m/Professionals_2024/assets/148451230/1aa398ac-dd3a-4887-b975-14ff7a3bf633)

Перезапускаем:
```
systemctl restart bind
```

</details>

## Настройте синхронизацию времени между сетевыми устройствами по протоколу NTP. 
a) В качестве сервера должен выступать SRV1-HQ 
1. Используйте стратум 5 
2. Используйте ntp2.vniiftri.ru в качестве внешнего сервера синхронизации времени

b) Все устройства должны синхронизировать своё время с SRV1-HQ. 
	1. Используйте chrony, где это возможно 

c) Используйте на всех устройствах московский часовой пояс.


<details>
  <summary>Настройте синхронизацию времени между сетевыми устройствами по протоколу NTP.</summary>

### 1. Установка Chrony на всех устройствах:
```
apt-get update && apt-get install chrony -y
```
### 2. Настройка NTP-сервера на SRV1-HQ:
Открываем конфигурационный файл:
```
vim /etc/chrony.conf
```
Вносим изменения:
```
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (https://www.pool.ntp.org/join.html).
# pool pool.ntp.org iburst
server ntp2.vniiftri.ru iburst

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Allow NTP client access from local network.
allow 0.0.0.0/0

# Serve time even if not synchronized to a time source.
local stratum 5
```
**Перезапуск Chrony:**
```
systemctl restart chronyd
systemctl enable chronyd
```
**Проверка статуса:**
```
chronyc tracking
chronyc sources
```

### 3. Настройка клиентов (всех остальных устройств):
На всех устройствах, кроме SRV1-HQ, редактируем конфигурацию:
```
vim /etc/chrony.conf
```
Изменяем настройки: (ip адрес может отличаться)
```
# Указываем IP SRV1-HQ
server 192.168.11.1 iburst # поправить тут

# Сохраняем сведения о состоянии времени
driftfile /var/lib/chrony/drift
makestep 1.0 3
```

**Перезапуск Chrony:**
```
systemctl restart chronyd
systemctl enable chronyd
```
**Проверка статуса:**
```
chronyc tracking
chronyc sources
```

### 4. Установка часового пояса:
На всех устройствах выполняем команду:
```
timedatectl set-timezone Asia/Yekaterinburg 
```
**Проверка:**
```bash
timedatectl status | grep 'Time zone:'
```
В выводе должно быть:
```
Time zone: Asia/Yekaterinburg (+05, +0500)
```
### 5. Проверка работы синхронизации:
**На сервере (SRV1-HQ):**
```
chronyc sources -v
```
Должна быть видна строка с `ntp2.vniiftri.ru`.

**На клиентах:**
```
chronyc sources -v
```
Должна быть видна строка с IP-адресом SRV1-HQ.

</details>


## Реализация доменной инфраструктуры SAMBA AD 
a) Сконфигурируйте основной доменный контроллер на SRV1-HQ
1. Используйте модуль BIND9_DLZ
2. Создайте 30 пользователей user1-user30 с паролем P@ssw0rd.
3. Пользователи user1-user10 должны входить в состав группы group1.
4. Пользователи user11-user20 должны входить в состав группы group2.
5. Пользователи user21-user30 должны входить в состав группы group3.
6. Создайте подразделения CLI и ADMIN
i. Поместите клиентов в подразделения в зависимости от их роли.
7. Клиентами домена являются ADMIN-DT, CLI-DT, ADMIN-HQ, CLI-HQ.
f) В качестве резервного контроллера домена используйте SRV1-DT.
1. Используйте модуль BIND9_DLZ
h) Реализуйте общую папку на SRV1-HQ 
1. Используйте название SAMBA
2. Используйте расположение /opt/data

<details>
    <summary>Реализация доменной инфраструктуры SAMBA AD </summary>

[Читай тут 1](https://www.altlinux.org/ActiveDirectory/DC)
[Читай тут 2](https://www.altlinux.org/SambaAD_start)

</details>
