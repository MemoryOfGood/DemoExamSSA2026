<img width="682" height="655" alt="Дэмо" src="https://github.com/user-attachments/assets/d2d81999-fb6d-42a3-bf05-70060b3137c6" />

**Рисунок 1**

Таблица 1

| Имя ВМ | Центральный процессор<br>(CPU) | Оперативная память<br>(RAM) | Накопитель,<br>Тип и объём | Операционная система<br>(тип для VMware)|
| ------ | --------------------------- | ------------------------ | ----------------------- | ------------------------------------------------ |
| ISP    | 1 ядро / 1 поток            | 1 ГБ (1024 МБ)           | SCSI, 25ГБ              | Alt Server 11<br>(Other Linux 6.x kernel 64-bit) |
| HQ-RTR | 1 ядро / 1 поток            | 1 ГБ (4096 МБ)           | IDE, 128 МБ               | RouterOS 7.22.1<br>(Other Linux 3.x kernel 64-bit) |
| BR-RTR | 1 ядро / 1 поток            | 1 ГБ (4096 МБ)           | IDE, 128 МБ               | RouterOS 7.22.1<br>(Other Linux 3.x kernel 64-bit) |
| HQ-SRV | 1 ядро / 1 поток            | 2 ГБ (2048 МБ)           | SCSI, 25ГБ              | Alt Server 11<br>(Other Linux 6.x kernel 64-bit) |
| BR-SRV | 1 ядро / 1 поток            | 2 ГБ (2048 МБ)           | SCSI, 25ГБ              | Alt Server 11<br>(Other Linux 6.x kernel 64-bit) |
| HQ-CLI | 1 ядро / 2 потока           | 1 ГБ (1024 МБ)           | SCSI, 25ГБ              | Alt Alt p11 Starterkit xfce<br>(Other Linux 6.x kernel 64-bit) |
| ИТОГО  | 7                           | ~ 8 ГБ (8192 МБ)       | ~ 101 ГБ                |                                                 |

# Модуль 1
## 1. Произведите базовую настройку устройств
* Настройте имена устройств согласно топологии. Используйте полное доменное имя 
* На всех устройствах необходимо сконфигурировать IPv4:
* IP-адрес должен быть из приватного диапазона, в случае, если сеть локальная, согласно RFC1918
* Локальная сеть в сторону HQ-SRV(VLAN 100) должна вмещать не более 32 адресов
* Локальная сеть в сторону HQ-CLI(VLAN 200) должна вмещать не менее 16 адресов
* Локальная сеть для управления(VLAN 999) должна вмещать не более 8 адресов
* Локальная сеть в сторону BR-SRV должна вмещать не более 16 адресов
* Сведения об адресах занесите в таблицу 2, в качестве примера используйте Прил_3_О1_КОД 09.02.06-1-2026-М1

> [!WARNING]
> На HQ-RTR и BR-RTR также надо добавить тетий адаптер "Только локальный"/"Host-Only" который будут использоваться в качестве управления через winbox

> [!NOTE]
> ISP будет настраиваться в следующем задании
### HQ-RTR
Задаём имя для роутера
```
system/identity/set name=HQ-RTR.au-team.irpo
```
Устанавливаем IP-адрес для маршутизатора
```
ip/address/add address=172.16.1.2/28 network=172.16.1.0 interface=ether1
```
Добавляем статический маршртут для выхода в сеть
```
ip/route/add dst-address=0.0.0.0/0 gateway=172.16.1.1
```
Создаём подинтерфесы для ether2
```
interface/vlan/add name=vlan100 vlan-id=100 interface=ether2
interface/vlan/add name=vlan200 vlan-id=200 interface=ether2
interface/vlan/add name=vlan999 vlan-id=999 interface=ether2
```

### BR-RTR 
Задаём имя для роутера
```
system/identity/set name=BR-RTR.au-team.irpo
```
Устанавливаем IP-адрес для маршутизатора
```
ip/address/add address=172.16.2.2/28 network=172.16.2.0 interface=ether1
ip/address/add address=192.168.3.1/28 network=192.168.3.0 interface=ether2
```
Добавляем статический маршртут для выхода в сеть
```
ip/route/add dst-address=0.0.0.0/0 gateway=172.16.2.1
```

### HQ-SRV
Задаём имя 
```bash
hostnamectl set-hostname HQ-SRV.au-team.irpo
```
Создаём папку для под-адаптера 
```bash
mkdir /etc/net/ifaces/ens33.100
```
Создаём файл
```bash
nano /etc/net/ifaces/ens33.100/options 
```
Записываем в него конфигурацию 
```bash
BOOTPROTO=static
TYPE=vlan
HOST=ens33
VID=100
CONFIG_IPV4=yes
DISABLED=no
```
Cохраняем (ctrl+x, y, enter)

Записываем IP-адрес и маршрут по умолчанию
```bash
echo “192.168.1.30/27” > /etc/net/ifaces/ens33.100/ipv4address
echo “default via 192.168.1.1” > /etc/net/ifaces/ens33.100/ipv4route
```
Перезапускаем службу отвечающую за сеть
```bash
systemctl restart network
```
Для проверки выведем IP-адреса
```bash
ip -c a
```

>[!Warning]
>Из-за того, что на Alt Linux могут пропадать IP-адреса после перезагрузки системы, добавим запись о перезапуске службы network

>[!TIP]
> Для удобства редактирования можно изменить стандартный редактор текста на nano
> ```bash
>  export EDITOR=nano 
>  ```

Переходим в crontab 
```bash
сrontab -e
```
Добавляем запись
```bash
@reboot /bin/systemctl restart network
```
Cохраняем (ctrl+x, y, enter)
### BR-SRV
Задаём имя 
```bash
hostnamectl set-hostname BR-SRV.au-team.irpo
```
Редактируем файл /etc/net/ifaces/ens33/options 
```bash
nano /etc/net/ifaces/ens33/options
```
Изменяем на статическую конфигурацию 
```bash
BOOTPROTO=static
DISABLED=no
TYPE=eth
CONFIG_IPV4=yes
```
Cохраняем (ctrl+x, y, enter)

Записываем IP-адрес и маршрут по умолчанию
```bash
echo “192.168.3.14/28” > /etc/net/ifaces/ens33/ipv4address
echo “default via 192.168.3.1” > /etc/net/ifaces/ens33/ipv4route 
```
Перезапускаем службу отвечающую за сеть
```bash
systemctl restart network
```
Выводим адреса для проверки
```bash
ip -c a
```
### HQ-CLI
Переходим в суперпользователя в терминале
```bash
su -
```
Задаём имя 
```bash
hostnamectl set-hostname HQ-CLI.au-team.irpo
```
Создаём vlan200 для ens33 с помощью nmcli
```
nmcli connection add type vlan con-name ens33.200 ifname ens33.200 dev ens33 id 200
```

### Таблица 2

| Имя устройства   | IP-адрес        | Шлюз по умолчанию |
| ---------------- | --------------- | ----------------- |
| HQ-RTR (ISP)     | 172.16.1.2/28   | 172.16.1.1        |
| - vlan 100 (SRV) | 192.168.1.1/27  | -                 |
| - vlan200 (CLI)  | 192.168.2.1/28  | -                 |
| - vlan999 (MGMT) | 192.168.99.1/29 | -                 |
| BR-RTR           | 172.16.2.2/28   | 172.16.2.1        |
| - (LAN)          | 192.168.3.1/28  |                   |
| HQ-SRV           | 192.168.1.30/27 | 192.168.1.1       |
| BR-SRV           | 192.168.3.14/28 | 192.168.3.1       |
| HQ-CLI           | DHCP            | 192.168.2.1       |

## 2. Настройте доступ к сети Интернет, на маршрутизаторе ISP
* Настройте адресацию на интерфейсах:
* Интерфейс, подключенный к магистральному провайдеру, получает адрес по DHCP
* Настройте маршрут по умолчанию, если это необходимо
* Настройте интерфейс, в сторону HQ-RTR, интерфейс подключен к сети 172.16.1.0/28
* Настройте интерфейс, в сторону BR-RTR, интерфейс подключен к сети 172.16.2.0/28
* На ISP настройте динамическую сетевую трансляцию портов для доступа к сети Интернет HQ-RTR и BR-RTR

Задаём имя 
```bash
hostnamectl set-hostname ISP.au-team.irpo
```
Адаптер ens34 (третий смотря по ip -c a)
Создаём или переписываем файл /etc/net/ifaces/ens34/options
```bash
nano /etc/net/ifaces/ens34/options
```
Конфигурация файла 
```bash
BOOTPROTO=static
DISABLED=no
TYPE=eth
CONFIG_IPV4=yes
```
Cохраняем (ctrl+x, y, enter)

Тоже самое проделывается для ens35 в /etc/net/ifaces/ens35/options
```bash
nano /etc/net/ifaces/ens35/options
```
Конфигурация файла
```bash
BOOTPROTO=static
DISABLED=no
TYPE=eth
CONFIG_IPV4=yes
```
Записываем адреса
```bash 
echo “172.16.1.1/28” > /etc/net/ifaces/ens34/ipv4address 
echo “172.16.2.1/28” > /etc/net/ifaces/ens35/ipv4address 
```
Перезапускаем службу отвечающую за сеть
```bash
systemctl restart network
```
Для проверки выводим IP-адреса
```bash
ip -c a
```
Переходим в crontab 
```bash
export EDITOR=nano
сrontab -e
```
Добавляем запись
```bash
@reboot /bin/systemctl restart network
```
Cохраняем (ctrl+x, y, enter)

Прописываем команду для маршрутизации внутри машины:
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```
Сохраним настройки маршрутизации в файле /etc/net/sysctl.conf
```bash
nano /etc/net/sysctl.conf
```
Изменяем строчку на net.ipv4.ip_forward=1 
```bash
net.ipv4.ip_forward=1
```
Cохраняем файл (ctrl+x, y, enter)

Создаём правило для iptables для маршрутизации во внешнюю сеть (NAT)
```bash
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens33 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens33 -j MASQUERADE
```
Сохраняем данные правила и добавляем iptables в автозагрузку
```bash
iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```
## 3. Создайте локальные учетные записи
* Создайте пользователя sshuser
* Пароль пользователя sshuser с паролем P@ssw0rd
* Идентификатор пользователя 2026
* Пользователь sshuser должен иметь возможность запускать sudo без ввода пароля
* Создайте пользователя net_admin на маршрутизаторах HQ-RTR и BR-RTR
* Пароль пользователя net_admin с паролем P@ssw0rd
* При настройке ОС на базе Linux, запускать sudo без ввода пароля
* При настройке ОС отличных от Linux пользователь должен обладать максимальными привилегиями.

### HQ-RTR и BR-RTR 
```
/user/add name=net_admin group=full password="P@ssw0rd"
```
### HQ-SRV и BR-SRV
```bash
useradd -m sshuser -u 2026 
passwd sshuser 
P@ssw0rd #Вводим пароль
P@ssw0rd #Вводим его второй раз для подтверждения
```
Редактируем файл /etc/sudoers
```bash
nano /etc/sudoers
```
>[!NOTE]
>nano может выводить сообщение, что файл только для чтения,
>но изменения сохраняются

Убираем комментарий у строчки
```
WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
```
Cохраняем файл (ctrl+x, y, enter)

Добавляем пользователя sshuser в группу wheel:
```bash
usermod -aG wheel sshuser
```
## 4. Настройте коммутацию в сегменте HQ (HQ-RTR)
* Трафик HQ-SRV должен принадлежать VLAN 100
* Трафик HQ-CLI должен принадлежать VLAN 200
* Предусмотреть возможность передачи трафика управления в VLAN 999
* Реализовать на HQ-RTR маршрутизацию трафика всех указанных VLAN с использованием одного сетевого адаптера ВМ/физического порта
* Сведения о настройке коммутации внесите в отчёт

```
ip/address/add address=192.168.1.1/27 network=192.168.1.0 interface=vlan100
ip/address/add address=192.168.2.1/28 network=192.168.2.0 interface=vlan200
ip/address/add address=192.168.99.1/29 network=192.168.99.0 interface=vlan999
```

## 5. Настройте безопасный удаленный доступ на серверах HQ-SRV и BR-SRV
* Для подключения используйте порт 2026
* Разрешите подключения исключительно пользователю sshuser
* Ограничьте количество попыток входа до двух 
* Настройте баннер «Authorized access only».
 
Редактируем /etc/openssh/sshd_config
```bash
nano /etc/openssh/sshd_config
```

Изменяем следующие параметры

>[!TIP]
> Чтобы найти определенные параметры можно использовать ```ctrl+w```
```bash
Port 2026
PermitRootLogin no
MaxAuthTries 2
Banner /etc/openssh/banner
```

Добавляем строчку
```bash
AllowUsers sshuser
```

Сохраняем и выходим из файла (crtl+x, y, enter)

Пишем баннер для ssh-server
```bash
echo “Authorized access only” > /etc/openssh/banner
```

После чего перезапускаем службу
```
systemctl restart sshd
```

## 6. Между офисами HQ и BR, на маршрутизаторах HQ-RTR и BR-RTR необходимо сконфигурировать ip туннель
* На выбор технологии GRE или IP in IP
* Сведения о туннеле занесите в отчёт

### HQ-RTR
```
interface/gre/add name=HQ-BR local-address=172.16.1.2 remote-address=172.16.2.2 allow-fast-path=no ipsec-secret="P@ssw0rd"
ip/address/add address=10.10.10.1/30 network=10.10.10.0 interface=HQ-BR
```
Для отчёта
```
interface/gre/print
```
### BR-RTR
```
interface/gre/add name=BR-HQ local-address=172.16.2.2 remote-address=172.16.1.2 allow-fast-path=no ipsec-secret="P@ssw0rd"
ip/address/add address=10.10.10.2/30 network=10.10.10.0 interface=BR-HQ
```
Для отчёта
```
interface/gre/print
```

## 7. Обеспечьте динамическую маршрутизацию на маршрутизаторах HQ-RTR и BR-RTR
* Сети одного офиса должны быть доступны из другого офиса и наоборот. Для обеспечения динамической маршрутизации используйте link state протокол на усмотрение участника
* Разрешите выбранный протокол только на интерфейсах ip туннеля
* Маршрутизаторы должны делиться маршрутами только друг с другом
* Обеспечьте защиту выбранного протокола посредством парольной защиты
* Сведения о настройке и защите протокола занесите в отчёт.
### HQ-RTR и BR-RTR
```
routing/ospf/instance/add
routing/ospf/area/add area-id=10.10.10.0 instance=ospf-instance-1
routing/ospf/interface-template/add area=ospf-area-1 networks="10.10.10.0/30, 192.168.1.0/27, 192.168.2.0/28, 192.168.3.0/28"
```

## 8. Настройка динамической трансляции адресов маршрутизаторах HQ-RTR и BR-RTR
* Настройте динамическую трансляцию адресов для обоих офисов в сторону ISP, все устройства в офисах должны иметь доступ к сети Интернет

### HQ-RTR и BR-RTR
```
ip/firewall/nat/add chain=srcnat action=masquerade out-interface=ether1
```

## 9. Настройте протокол динамической конфигурации хостов для сети в сторону HQ-CLI
* Настройте нужную подсеть 
* В качестве сервера DHCP выступает маршрутизатор HQ-RTR
* Клиентом является машина HQ-CLI
* Исключите из выдачи адрес маршрутизатора
* Адрес шлюза по умолчанию – адрес маршрутизатора HQ-RTR
* Адрес DNS-сервера для машины HQ-CLI – адрес сервера HQ-SRV
* DNS-суффикс – au-team.irpo
* Сведения о настройке протокола занесите в отчёт.

### HQ-RTR
Cоздаём пул адресов
```
ip/pool/add name=hq-pool ranges="192.168.2.2-192.168.2.14"
```
Cоздаем сервер, указывая интерфейс vlan200 и пул hq-pool
```
ip/dhcp-server/add name=dhcp-server interface=vlan200 address-pool=hq-pool
```
Указываем параметры для DHCP-сервера
```
ip/dhcp-server/network/add address=192.168.2.0/28 gateway=192.168.2.1 netmask=28 dns-server=192.168.1.30 domain="au-team.irpo"
```

### HQ-CLI
Переходим в суперпользователя в терминале
```bash
su -
```
Выполняем запуск dhcp-клиента
```bash
dhcpcd
```
Выводим адреса для проверки
```bash
ip -c a
```

## 10. Настройте инфраструктуру разрешения доменных имён для офисов HQ и BR
* Основной DNS-сервер реализован на HQ-SRV
* Сервер должен обеспечивать разрешение имён в сетевые адреса устройств и обратно в соответствии с таблицей 3
* В качестве DNS сервера пересылки используйте любой общедоступный DNS сервер (77.88.8.7, 77.88.8.3 или другие)


**Таблица 3**

| Устройство                                    | Запись              | Тип    | IP-адрес                          |
| --------------------------------------------- | ------------------- | ------ | --------------------------------- |
| HQ-RTR                                        | hq-rtr.au-team.irpo | A, PTR | 192.168.1.1                       |
| BR-RTR                                        | br-rtr.au-team.irpo | A      | 192.168.3.1                       |
| HQ-SRV                                        | hq-srv.au-team.irpo | A, PTR | 192.168.1.30                      |
| HQ-CLI                                        | hq-cli.au-team.irpo | A, PTR | 192.168.2.14<br>(Смотрите по DHCP) |
| BR-SRV                                        | br-srv.au-team.irpo | A      | 192.168.3.14                      |
| ISP (интерфейс направленный в сторону HQ-RTR) | docker.au-team.irpo | A      | 172.16.1.1                        |
| ISP (интерфейс направленный в сторону BR-RTR) | web.au-team.irpo    | A      | 172.16.2.1                        |

Прописываем общедоступный DNS-сервер 
```bash
echo nameserver 77.88.8.8 > /etc/net/ifaces/ens33.100/resolv.conf
```

Перезапускаем сеть
```bash
service network restart
```

Проверяем на наличие пакета dnsmasq
```
apt-cache policy dnsmasq
```

> [!NOTE]
> Если не установлен, устанавливаем пакет dnsmasq
>```bash
> apt-get update
> apt-get install dnsmasq
>```

Редактируем конфигурационный файл dnsmasq.conf
```bash
nano /etc/dnsmasq.conf
```

>[!Warning]
> Перед редактирование проверьте ip-адрес HQ-CLI

И добавляем в неё строки (для удобства прям с первой строки файла):
```bash
no-resolv
domain=au-team.irpo
#server=/au-team.irpo/192.168.3.14
server=77.88.8.8
interface=*

#HQ-RTR                                     
address=/hq-rtr.au-team.irpo/192.168.1.1
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo
address=/hq-rtr.au-team.irpo/192.168.2.1
ptr-record=1.2.168.192.in-addr.arpa,hq-rtr.au-team.irpo
#BR-RTR
address=/br-rtr.au-team.irpo/192.168.3.1
#HQ-SRV
address=/hq-srv.au-team.irpo/192.168.1.30
ptr-record=30.1.168.192.in-addr.arpa,hq-srv.au-team.irpo
#HQ-CLI 
address=/hq-cli.au-team.irpo/192.168.2.14
ptr-record=2.2.168.192.in-addr.arpa,hq-cli.au-team.irpo
#BR-SRV
address=/br-srv.au-team.irpo/192.168.3.14
#web
address=/web.au-team.irpo/172.16.1.1
#docker
address=/docker.au-team.irpo/172.16.2.1
```

Сохраняем и выходим из файла (ctrl+x, y enter)

Редактируем файл hosts
```bash
nano /etc/hosts
```
Добавляем адрес сервера
```bash
192.168.1.30   HQ-SRV.au-team.irpo
```

Сохраняем и выходим из файла (ctrl+x, y enter)

Перезапускаем ДНС-сервер
```bash
systemctl restart dnsmasq
```

Добавляем строчку в crontab
```bash
export EDITOR=nano
сrontab -e
```

И вносим в конец файла следующую строчку:
```bash
@reboot /bin/systemctl restart dnsmasq
```

### BR-SRV

Прописываем настроенный DNS-сервер 
```bash
echo nameserver 192.168.1.30 | tee /etc/net/ifaces/ens33/resolv.conf
echo domain au-team.irpo | tee -a /etc/net/ifaces/ens33/resolv.conf
```
Перезапускаем сеть для обновления ДНС
```bash
systemctl restart network
```
Сохраняем и выходим из файла (ctrl+x, y enter)


### ISP, HQ-CLI и BR-SRV
> [!Warning]
> Выполните обновление репозиториев один раз после этого пункта, чтобы позже не обновлять их
> ```
> apt-get update
> ```

## 11. Настройте часовой пояс на всех устройствах 
* за исключением виртуального коммутатора, в случае его использования
* согласно месту проведения экзамена

### HQ-RTR, BR-RTR
```
system/clock/set time-zone-name=Asia/Irkutsk
```
### ISP, HQ-SRV, HQ-CLI, BR-SRV
Выключаем службу chrony:
```bash
systemctl disable --now chronyd
```
Cкачиваем службу systemd-timesyncd:
```bash
apt-get install systemd-timesyncd
```
Теперь включим службу systemd-timesyncd и посмотрим её статус работы:
```bash
systemctl enable --now systemd-timesyncd
timedatectl set-timezone Asia/Irkutsk
timedatectl timesync-status
```
# Модуль 2. Организация сетевого администрирования

## 1. Настройте контроллер домена Samba DC на сервере BR-SRV
* Имя домена au-team.irpo
* Введите в созданный домен машину HQ-CLI
* Создайте 5 пользователей для офиса HQ: имена пользователей формата hquser№ (например hquser1, hquser2 и т.д.)
* Создайте группу hq, введите в группу созданных пользователей
* Убедитесь, что пользователи группы hq имеют право аутентифицироваться на HQ-CLI
* Пользователи группы hq должны иметь возможность повышать привилегии для выполнения ограниченного набора команд: cat, grep, id. Запускать другие команды с повышенными привилегиями пользователи группы права не имеют.

### HQ-SRV
Редактируем конфигурацию нашего DNS-сервера на HQ-SRV
```bash
nano /etc/dnsmasq.conf
```
Убираем комментарий со строчки
```bash
server=/au-team.irpo/192.168.3.14
```
Перезапускаем dnsmasq как службу:
```bash
systemctl restart dnsmasq
```

### BR-SRV
Для Samba DC (на базе Heimdal Kerberos) необходимо установить компонент samba-dc:
```bash
apt-get install task-samba-dc
```

Необходимо очистить базы и конфигурацию Samba (домен, если он создавался до этого, будет удалён):
```bash
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol
```

>[!Warning]
> Обязательно удаляйте /etc/samba/smb.conf перед созданием домена:
>```bash
> rm -f /etc/samba/smb.conf
>```

Для того чтобы работал домен проверяем наличие и правильность ДНС в файле /etc/resolv.conf:
```bash
cat /etc/resolv.conf
```
Вывод файла /etc/resolv.conf
```bash
nameserver 192.168.1.30
domain au-team.irpo
```

Для интерактивного развертывания запустите samba-tool domain provision, это запустит утилиту развертывания, которая будет задавать различные вопросы о требованиях к установке.
```bash
samba-tool domain provision
```

При запросе ввода нажимайте **Enter** за исключением запроса пароля администратора («Administrator password:» и «Retype password:»).
>[!TIP]
> Лучше исползовать стандартный пароль, который был указан ранее в задании
> ```P@ssw0rd```

У Samba есть свой собственный DNS-сервер. В DNS forwarder IP address нужно проверить что указывается/указать DNS-сервер HQ-SRV (192.168.1.30), чтобы DC мог разрешать внешние доменные имена.

После удачного развертывания домена выведет сообщение об этом:

**Рисунок**

Перемещаем сгенерированный конфиг krb5.conf:
```bash
mv -f /var/lib/samba/private/krb5.conf /etc/krb5.conf
```
Устанавливаем службу по умолчанию и запустить её:
```bash
systemctl enable --now samba
```

Повторяем настройку автозагрузки в crontab
```bash
export EDITOR=nano
сrontab -e
```
И вносим в конец файла следующие строки:
```bash
@reboot /bin/systemctl restart samba
```

Теперь **ПЕРЕЗАПУСКАЕМ** машину BR-SRV:
```bash
reboot
```

Проверяем работу домена:
```bash
samba-tool domain info 127.0.0.1
```

>[!TIP]
>Если команда не работает, то перед этим стоит перезапустить пакет samba:
>```bash
>systemctl restart samba
>```


Cоздадим 5 пользователей:
```bash
samba-tool user add user1.hq Student1
samba-tool user add user2.hq Student2
samba-tool user add user3.hq Student3
samba-tool user add user4.hq Student4
samba-tool user add user5.hq Student5
```

Создадим группу и поместим туда созданных пользователей:
```bash
samba-tool group add hq
samba-tool group addmembers hq user1.hq,user2.hq,user3.hq,user4.hq,user5.hq
```
### HQ-CLI

Устанавливаем пакет, отвечающий за подключение в домен task-auth-ad-sssd:
```bash
apt-get install task-auth-ad-sssd
```

Вызываем ЦУС (Центр управления системой)
```
acc
```

В **Центре управления системой** в разделе **Пользователи** выбираем **Аутентификация**.

В открывшемся окне следует: 
выбираем пункт **Домен Active Directory**, 
вводим домен **au-team.irpo**,
рабочую группу **au-team**,
ставим галочку у **восстановить файлы…**
После нажимаем кнопку **Применить**:

<img width="835" height="549" alt="Снимок экрана 2025-12-17 140616" src="https://github.com/user-attachments/assets/25214522-4242-4ad9-87ff-6715ab0e4507" />

**Рисунок** 

Соглашаемся на восстановление файлов конфигурации по умолчанию
<img width="422" height="171" alt="Снимок экрана 2025-12-17 140713" src="https://github.com/user-attachments/assets/3984cb32-8cf4-4bcc-ba24-e83309850dde" />
**Рисунок**

Вводим пароль администратора, который ввели при создания домена (P@ssw0rd) и нажимаем кнопку **ОК**:
<img width="476" height="286" alt="Снимок экрана 2025-12-17 141118" src="https://github.com/user-attachments/assets/1584875d-c8c7-4e77-b881-e8785e023c44" />

**Рисунок** 

При успешном вводе в домен выведется информация:
<img width="236" height="151" alt="Снимок экрана 2025-12-17 141156" src="https://github.com/user-attachments/assets/b5dfea0a-1508-4f0b-85da-8cfb27fe94a6" />

**Рисунок** 

Перезагружаем систему командой ```reboot``` или через графический интерфейс
## 2. Сконфигурируйте файловое хранилище на сервере HQ-SRV
* При помощи двух подключенных к серверу дополнительных дисков размером 1 Гб сконфигурируйте дисковый массив уровня 0
* Имя устройства – md0, при необходимости конфигурация массива размещается в файле /etc/mdadm.conf 
* Создайте раздел, отформатируйте раздел, в качестве файловой системы используйте ext4 
* Обеспечьте автоматическое монтирование в папку /raid

Перед начало выключаем ВМ и добавляем 2 виртуальных жестких дисках по 1 ГБ в VmWare Workstation для HQ-SRV  
Переходим **Edit virtual machine settings** > **Add..** > **Hard Disk**

<img width="378" height="332" alt="Снимок экрана 2025-11-10 080737" src="https://github.com/user-attachments/assets/6d1a8020-f0dd-43bf-82de-4d9b2be3d17c" />

**Рисунок** 

Просматриваем имеющийся диски и запоминаем их имена
```bash
lsblk
```
<img width="518" height="537" alt="изображение" src="https://github.com/user-attachments/assets/aa88603f-40f9-47f0-8a03-6f23989d4640" />

**Рисунок** 

Создаём raid-массив и форматируем его в файловой системе ext4
```bash
mdadm --create --verbose /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc
mkfs.ext4 -F /dev/md0
```

Создаём папку и создаём точку подключения
```bash
mkdir /raid
mount /dev/md0 /raid
```

Просмотр точек подключения
```bash
df -h
```
<img width="643" height="180" alt="Снимок экрана 2025-11-10 085400" src="https://github.com/user-attachments/assets/c73a0b06-c9af-4432-913a-2de91099a4b5" />

**Рисунок**

Считываем характеристики массива и записываем в файл /etc/mdadm/mdadm.conf, чтобы массив автоматически подключался после перезагрузки
```bash
mdadm --detail --scan | tee -a /etc/mdadm.conf
echo '/dev/md0 /raid ext4 defaults,nofail,discard 0 0' | tee -a /etc/fstab
```

## 3. Настройте сервер сетевой файловой системы (nfs) на HQ-SRV
* В качестве папки общего доступа выберите /raid/nfs, доступ для чтения и записи исключительно для сети в сторону HQ-CLI
* На HQ-CLI настройте автомонтирование в папку /mnt/nfs
* Основные параметры сервера отметьте в отчёте

### HQ-RTR
>[!WARNING]
>Выполняем проброс портов чтобы можно было подключиться по nfs к HQ-SRV
```
ip/firewall/nat/add chain=dstnat action=dst-nat protocol=tcp port=2049 to-ports=2049
ip/firewall/nat/add chain=dstnat action=dst-nat protocol=udp port=2049 to-ports=2049
```

### HQ-SRV
Cкачиваем пакет nfs-server
```bash
apt-get install nfs-server
```

Создаём папку, которая будет использоваться как сетевая папка и выдаём особые права:

```bash
mkdir /raid/nfs
chown 99:99 /raid/nfs
chmod 777 /raid/nfs
```

Прописываем параметры для того что можно было подключится к сетевой папке и принимаем их:
```bash
echo '/raid/nfs 192.168.2.0/28(rw,sync,no_subtree_check)' | tee -a /etc/exports
exportfs -rav
exportfs -v
```
<img width="829" height="100" alt="Снимок экрана 2025-11-10 091311" src="https://github.com/user-attachments/assets/d4921e15-552b-4473-8b0d-80b32f4c1c78" />

**Рисунок**


Включаем и перезапускаем службу NFS:
```bash
systemctl enable --now nfs
systemctl restart nfs
```
### HQ-CLI

Cкачиваем пакет nfs-clients
```bash
apt-get install nfs-clients
```

Создадим папку куда будем монтировать сетевую папку
```bash
mkdir -p /mnt/nfs
```

Вносим строку с подключением сетевой папки в fstab:
```bash
echo 'HQ-SRV:/raid/nfs /mnt/nfs nfs intr,soft,_netdev,x-systemd.automount 0 0' | tee -a /etc/fstab
```
Подключаем и проверяем что работает:
```bash
mount -a
df -h
```
<img width="905" height="282" alt="Снимок экрана 2025-11-10 094047" src="https://github.com/user-attachments/assets/050f40dc-9b11-4bed-808a-5f01280ad6da" />

**Рисунок**

## 4.  Настройте службу сетевого времени на базе сервиса chrony на маршрутизаторе ISP
* Вышестоящий сервер ntp на маршрутизаторе ISP - на выбор участника 
* Стратум сервера - 5
* В качестве клиентов ntp настройте: HQ-SRV, HQ-CLI, BR-RTR, BR-SRV

### ISP
Eстанавливаем пакет chrony

```bash
apt-get install chrony
```

Проверяем статус служб
```bash
systemctl status chrony
timedatectl
```

Редактируем файл /etc/chrony.conf
```bash
nano /etc/chrony.conf
```

Закомментировать 2 строчки
```bash
#pool pool.ntp.org iburst

#rtcsync
```

Прописываем стратум, IP-адреса сетей которые могут обращаться к серверу и NTP-сервера
```bash
pool 0.ru.pool.ntp.org iburst
pool 1.ru.pool.ntp.org iburst
pool 2.ru.pool.ntp.org iburst
pool 3.ru.pool.ntp.org iburst

local stratum 5
allow 192.168.1.0/27
allow 192.168.2.0/28
allow 192.168.3.0/28
allow 172.16.1.0/28
allow 172.16.2.0/28
```

Включаем и перезапускаем службу chrony:
```bash
systemctl enable --now chrony
systemctl restart chrony
```

### HQ-RTR
Сначало создадим правило для порта 123 для подключения к NTP-серверу
```
ip/firewall/filter/add action=accept chain=input protocol=udp port=123
```
```
system/ntp/client/servers/add address=172.16.1.1
system/ntp/client/edit enabled
```
Прописываем в файле
```
yes
```
Сохраняем
```
ctrl+o
```

### BR-RTR
Сначало создадим правило для порта 123 для подключения к NTP-серверу
```
ip/firewall/filter/add action=accept chain=input protocol=udp port=123
```
```
system/ntp/client/servers/add address=172.16.2.1
system/ntp/client/edit enabled
```
Прописываем в файле
```
yes
```
Сохраняем
```
ctrl+o
```
### HQ-CLI и HQ-SRV
Изменяем в конфиге: /etc/systemd/timesyncd.conf
```bash
nano /etc/systemd/timesyncd.conf
```
Указываем в качестве сервера времени ISP 
```bash
[Time]
NTP=172.16.1.1
```
Сохраняем файл (ctrl+x, y, enter)

Теперь перезапускаем службу systemd-timesyncd и посмотрим её статус работы:
```bash
systemctl restart systemd-timesyncd
timedatectl timesync-status
```
### BR-SRV
Изменяем в конфиге: /etc/systemd/timesyncd.conf
```bash
nano /etc/systemd/timesyncd.conf
```
Указываем в качестве сервера времени ISP 
```bash
[Time]
NTP=172.16.2.1
```
Сохраняем файл (ctrl+x, y, enter)

Теперь перезапускаем службу systemd-timesyncd и посмотрим её статус работы:
```bash
systemctl restart systemd-timesyncd
timedatectl timesync-status
```

## 5. Сконфигурируйте ansible на сервере BR-SRV:
* Сформируйте файл инвентаря, в инвентарь должны входить HQ-SRV, HQ-CLI, HQ-RTR и BR-RTR 
* Рабочий каталог ansible должен располагаться в /etc/ansible 
* Все указанные машины должны без предупреждений и ошибок отвечать pong на команду ping в ansible посланную с BR-SRV.

### HQ-RTR и BR-RTR
Перед началом запускаем ssh-сервер 
```
/ip service enable ssh
```
### HQ-CLI
Перед началом установим проверяем наличие ssh-сервера 
```bash
su -
systemctl status sshd
```

>[!NOTE]
>Если отсутствует, то устанавливаем
>```bash
> apt-get install openssh-server
>```

>[!Note]
>Если не включен ```inactive (dead)```, то включаем
>```
> systemctl enable --now sshd
>```

### BR-SRV
Устанавливаем пакет ansible
```bash
apt-get install ansible
```

Устанавливаем плагины для работы с микротиком
```
ansible-galaxy collection install ansible.netcommon
ansible-galaxy collection install community.routeros
```

Редактируем файл hosts
```bash
nano /etc/ansible/hosts
```
>[!TIP]
>Указываем для HQ-SRV пользователя которого создавали в 3 задании 1 модуля,
>для HQ-CLI пользователя который был создан при установке (в данном случае user:1)

```bash
[RTR]
hq-rtr ansible_host=hq-rtr
br-rtr ansible_host=br-rtr
[RTR:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
ansible_user=net_admin
ansible_password=P@ssw0rd
[SRV]
hq-srv ansible_host=hq-srv
[SRV:vars]
ansible_user=sshuser
ansible_password=P@ssw0rd
ansible_port=2026
[CLI]
hq-cli ansible_host=hq-cli
[CLI:vars]
ansible_user=user
ansible_password=1
```
Сохраняем файл (ctrl+x, y, enter)

Записываем конфигурацию в файл
```bash
nano /etc/ansible/ansible.cfg
```
```bash
[defaults]
ansible_python_interpreter = /usr/bin/python3
inventoty = /etc/ansible/hosts
host_key_checking = false
```
Сохраняем и выходим из файла (crtl+x, y, enter)

Выполняем pong для всех устройств
```bash
ansible all -m ping
```
## 6. Разверните веб приложение в docker на сервере BR-SRV:
* Средствами docker должен создаваться стек контейнеров с веб приложением и базой данных
* Используйте образы site_latest и mariadb_latest располагающиеся в директории docker в образе Additional.iso
* Основной контейнер testapp должен называться tespapp
* Контейнер с базой данных должен называться db 
* Импортируйте образы в docker, укажите в yaml файле параметры подключения к СУБД, имя БД - testdb, пользователь testс паролем P@ssw0rd, порт приложения 8080, при необходимости другие параметры
* Приложение должно быть доступно для внешних подключений через порт 8080
### BR-SRV

Обновляем репозитории и устанавливаем docker-compose-v2
```bash
apt-get update
apt-get install docker-engine docker-compose-v2
```
Добавляем в автозагрузку и смотрим статус docker:
```bash
systemctl enable -–now docker
systemctl status docker
```

Подключаем [образ](https://disk.yandex.ru/d/0MGlkrp2B9nXDw) в VmWare workstation

<img width="702" height="414" alt="Pasted image 20251222113428" src="https://github.com/user-attachments/assets/9752a3f0-651a-4c1c-839b-1a967d5f4f22" />

**Рисунок**

Создаём папку для подключения
```
mkdir /media/ALTLinux
```

Монтируем подключаемый образ:
```bash
mount /dev/cdrom /media/ALTLinux
```
Просматриваем содержимое:
```bash
ls -lah /media/ALTLinux
```
<img width="516" height="164" alt="Pasted image 20251222114402" src="https://github.com/user-attachments/assets/bbce3328-ef24-42ff-a04d-f7ee5e18f0d6" />

**Рисунок**

Загружаем образы в систему
```bash
docker load < /media/ALTLinux/docker/site_latest.tar
docker load < /media/ALTLinux/docker/mariadb_latest.tar
```
Выводим список образов docker’а
```bash
docker images
```

<img width="747" height="100" alt="Pasted image 20251222114543" src="https://github.com/user-attachments/assets/67f414d1-340b-41cb-8767-6d957f47015f" />

**Рисунок**

Создаём файл docker-compose.yml
```bash
nano docker-compose.yml
```
>[!WARNING]
>Внимательно следите за отступами, оступы делаются пробелом, а не "TAB"
```yaml
services:
 testapp:
  image: site:latest
  container_name: testapp
  restart: always
  depends_on:
    - db
  ports:
    - "8080:8000"
  environment:
    DB_TYPE: maria
    DB_HOST: db
    DB_NAME: testdb
    DB_PORT: 3306
    DB_USER: test
    DB_PASS: P@ssw0rd
db:
  image: mariadb:10.11
  container_name: db
  restart: always
  environment:
    MYSQL_ROOT_PASSWORD: P@ssw0rd
    MYSQL_DATABASE: testdb
    MYSQL_USER: test
    MYSQL_PASSWORD: P@ssw0rd
  volumes:
    - db_data:/var/lib/mysql

volumes:
  db_data:
```
Сохраняем и выходим из файла (crtl+x, y, enter)

Запускаем docker
```bash
docker compose up -d
```

### HQ-CLI
Заходим в Chromium и вводим адрес 
```http
192.168.3.14:8080
```

<img width="974" height="204" alt="Pasted image 20251222123235" src="https://github.com/user-attachments/assets/20c8b486-d6f1-40de-b5aa-8556514abd5d" />

**Рисунок**

<img width="936" height="375" alt="Pasted image 20251222123457" src="https://github.com/user-attachments/assets/692c1b2a-1c30-475a-a3b0-6bc357a16fa1" />

**Рисунок**

## 7.Разверните веб приложение на сервере HQ-SRV:
* Используйте веб-сервер apache
* В качестве системы управления базами данных используйте mariadb 
* Файлы веб приложения и дамп базы данных находятся в директории web образа Additional.iso
* Выполните импорт схемы и данных из файла dump.sql в базу данных webdb
* Создайте пользователя web с паролем P@ssw0rd и предоставьте ему права доступа к этой базе данных
* Файлы index.php и директорию images скопируйте в каталог веб сервера apache
* В файле index.php укажите правильные учётные данные для подключения к БД
* Запустите веб сервер и убедитесь в работоспособности приложения
* Основные параметры отметьте в отчёте

### HQ-SRV
Устанавливаем необходимые пакеты для работы веб-сервиса (apache, mariadb и php)
```bash
apt-get install lamp-server
```

Добавляем службы в автозагрузку
```bash
systemctl enable --now httpd2
systemctl enable --now mariadb
```

Подключаем [образ](https://disk.yandex.ru/d/0MGlkrp2B9nXDw) в VmWare workstation

<img width="702" height="414" alt="Pasted image 20251222113428" src="https://github.com/user-attachments/assets/30fca107-94eb-48f2-a443-00438defebc1" />

**Рисунок**

Создаём папку для подключения
```
mkdir /media/ALTLinux
```
Монтируем подключаемый образ:
```bash
mount /dev/cdrom /media/ALTLinux
```
Просматриваем содержимое:
```bash
ls -lah /media/ALTLinux
```

<img width="412" height="131" alt="Pasted image 20251222125424" src="https://github.com/user-attachments/assets/81b1cd0a-d484-4612-a970-c899a740c5c8" />

**Рисунок**

Копируем содержимое папки web с диска в директорию /var/www/html:
```bash
cp /media/web/index.php /var/www/html
cp /media/web/logo.png /var/www/html
```

Редактируем файл index.php
```bash
nano /var/www/html/index.php
```
>[!NOTE]
>nano может выводить сообщение, что файл только для чтения,
>но изменения сохраняются

Изменяем параметры на те которые указанные в задании и которые были заданы ранее
```bash
$servername = "localhost";
$username = "web"
$password = "P@ssw0rd";
$dbname = "webdb";
```

<img width="713" height="108" alt="Pasted image 20251222212410" src="https://github.com/user-attachments/assets/70d9cce6-d791-40eb-9b13-fa4ef581ba06" />

**Рисунок**

Сохраняем и выходим из файла (ctrl+x, y, enter) 

Запускаем и добавляем в автозагрузку mariadb
```bash
systemctl enable --now mariadb
```

Заходим в консоль mariadb
```bash
mariadb
```
Создаем базу данных
```SQL
CREATE DATABASE webdb;
```
Создаём пользователя web
```SQL
CREATE USER web@localhost IDENTIFIED BY 'P@ssw0rd';
```
Выдаём права доступа этому пользователю на базу данных web и применяем
```SQL
GRANT ALL PRIVILEGES ON webdb.* TO web@localhost WITH GRANT OPTION;
```
Выходим из консоли mariadb
```SQL
exit
```
Импортируем схему данных из файла dump.sql, из папки web
```bash
mariadb -u web -pP@ssw0rd webdb < /media/web/dump.sql
```
Снова заходим в консоль mariadb
```bash
mariadb webdb
```
Выводим таблицы для проверки импорта
```SQL
SHOW TABLES;
```
Выходим из консоли mariadb
```SQL
exit
```
Запускаем и добавляем в автозагрузку apache2
```bash
systemctl enable --now httpd2
```

### HQ-CLI
Заходим в Chromium и вводим адрес
```http
192.168.1.30
```

## 8. На маршрутизаторах сконфигурируйте статическую трансляцию портов: 
* Пробросьте порт 8080 в порт приложения testapp BR-SRV на маршрутизаторе BR-RTR, для обеспечения работы приложения testapp извне
* Пробросьте порт 8080 в порт веб приложения на HQ-SRV на маршрутизаторе HQ-RTR, для обеспечения работы веб приложения извне
* Пробросьте порт 2026 на маршрутизаторе HQ-RTR в порт 2026 сервера HQ-SRV, для подключения к серверу по протоколу ssh из внешних сетей
* Пробросьте порт 2026 на маршрутизаторе BR-RTR в порт 2026 сервера BR-SRV, для подключения к серверу по протоколу ssh из внешних сетей. 

### HQ-RTR
```
ip/firewall/nat/add chain=dstnat dst-address=172.16.1.2 protocol=tcp port=2026 action=dst-nat to-addresses=192.168.1.30 to-ports=2026
ip/firewall/nat/add chain=dstnat dst-address=172.16.1.2 protocol=tcp port=8080 action=dst-nat to-addresses=192.168.1.30 to-ports=80
```

### BR-RTR
```
ip/firewall/nat/add chain=dstnat dst-address=172.16.2.2 protocol=tcp port=2026 action=dst-nat to-addresses=192.168.3.14 to-ports=2026
ip/firewall/nat/add chain=dstnat dst-address=172.16.2.2 protocol=tcp port=8080 action=dst-nat to-addresses=192.168.3.14 to-ports=8080
```

## 9. Настройте веб-сервер nginx как обратный прокси-сервер на ISP
* При обращении по доменному имени web.au-team.irpo у клиента должно открываться веб приложение на HQ-SRV
* При обращении по доменному имени docker.au-team.irpo у клиента должно открываться веб приложение testapp

### ISP
```
nano /etc/nginx/nginx.conf
```
Вносим свою конфигурацию
```bash
http {
	server {
	    listen 172.16.1.1:80;
	    server_name web.au-team.irpo;
		#auth_basic "Web-authentication";
	    #auth_basic_user_file /etc/nginx/.htpasswd;
	    location / {
	        proxy_pass http://172.16.1.2:8080;
	        proxy_set_header Host $host;
	        proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
			proxy_set_header X-Forwarded-Proto $scheme;
	    }
	}
	server {
	    listen 172.16.2.1:80;
	    server_name docker.au-team.irpo;
	    location / {
	        proxy_pass http://172.16.2.2:8080;
	        proxy_set_header Host $host;
	        proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
			proxy_set_header X-Forwarded-Proto $scheme;
	    }
	}
}
```
Сохраняем и выходим

```bash
systemctl enable --now nginx
```
### HQ-CLI
Заходим в Chromium и вводим адреса

Первая вкладка
```http
web.au-team.irpo
```

Вторая вкладка
```http
docker.au-team.irpo
```

## 10. На маршрутизаторе ISP настройте web-based аутентификацию:
* При обращении к сайту web.au-team.irpo клиенту должно быть предложено ввести аутентификационные данные
* В качестве логина для аутентификации выберите WEB с паролем P@ssw0rd
* Выберите файл /etc/nginx/.htpasswd в качестве хранилища учётных записей
* При успешной аутентификации клиент должен перейти на веб сайт.

### ISP
Устанавливаем пакет  
```bash
apt install apache-utils
```
Создаём пользователя для аутентификации с командой htpasswd и вводим дважды пароль **P@ssw0rd**
```bash
htpasswd -c /etc/nginx/.htpasswd WEB
```
Редактируем файл
```bash
nano /etc/nginx/nginx.conf
```
Убираем комментарии со строчек
```bash
auth_basic "Web-authentication";
auth_basic_user_file /etc/nginx/.htpasswd;
```
Перезапускаем nginx
```bash
systemctl restart nginx
```
### HQ-CLI 
Заходим в Chromium и вводим адрес
```http
web.au-team.irpo
```

## 11. Удобным способом установите приложение Яндекс Браузер на HQ-CLI 
* Установку браузера отметьте в отчёте.

Устанавливаем Яндекс Браузер
```bash
apt-get install yandex-browser-stable
```

<img width="1009" height="316" alt="Pasted image 20251223212809" src="https://github.com/user-attachments/assets/e1f1b1fc-d25d-4f9c-ab20-960eab73e298" />

**Рисунок**
