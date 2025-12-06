<img width="463" height="593" alt="Снимок экрана 2025-11-24 130720" src="https://github.com/user-attachments/assets/8740b3f1-048e-48d5-a7bf-252bf3cf63d9" />

**Рисунок 1**

Таблица 1

| Имя ВМ | Центральный процессор (CPU) | Оперативная память (RAM) | Накопитель, Тип и объём | Операционная система (тип для VMware)                |
| ------ | --------------------------- | ------------------------ | ----------------------- | -----------------------------------------------------|
| ISP    | 1 ядро / 1 поток            | 1 ГБ (1024 МБ)           | SCSI, 25ГБ              | Alt Server 11 (Other Linux 6.x kernel 64-bit)        |
| HQ-RTR | 1 ядро / 1 поток            | 4 ГБ (4096 МБ)           | IDE, 8 ГБ               | EcoRouter (Debian 10.x 64-bit)                       |
| BR-RTR | 1 ядро / 1 поток            | 4 ГБ (4096 МБ)           | IDE, 8 ГБ               | EcoRouter (Debian 10.x 64-bit)                       |
| HQ-SRV | 1 ядро / 1 поток            | 2 ГБ (2048 МБ)           | SCSI, 25ГБ              | Alt Server 11 (Other Linux 6.x kernel 64-bit)        |
| BR-SRV | 1 ядро / 1 поток            | 2 ГБ (2048 МБ)           | SCSI, 25ГБ              | Alt Server 11 (Other Linux 6.x kernel 64-bit)        |
| HQ-CLI | 1 ядро / 2 потока           | 1 ГБ (1024 МБ)           | SCSI, 25ГБ              | Alt Starterkits Xfce (Other Linux 6.x kernel 64-bit) |
| ИТОГО  | 7                           | ~ 14 ГБ (14336 МБ)       | ~ 116 ГБ                |                                                      |

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

> [!NOTE]
> ISP будет настраиваться в следующем задании
### HQ-RTR
Переходим к режиму администрирования
```
en
```

Переходим в конфигурационный режим
```
config
```

Задаём имя для роутера
```
hostname HQ-RTR.au-team.irpo
```

>[!Tip]
>Для выхода из этого режима или с любого подуровня конфигурации используется команда exit или сочетания клавиш Ctrl+D 

Создаем интерфейсы который присоединим к портам позже
```
interface ISP
    ip nat outside
    ip address 172.16.1.2/28
	no shutdown
	ctrl+d
interface SRV
	ip mtu 1500
	ip nat inside
	ip address 192.168.1.1/27
	no shutdown
	ctrl+d
interface CLI
	ip mtu 1500
	ip nat inside
	ip address 192.168.2.1/28
	no shutdown
	ctrl+d
interface MGMT
	ip mtu 1500
	ip nat inside
	ip address 192.168.99.1/29
	no shutdown
	ctrl+d
```

Присоединяем интерфейс ISP к порту ge0
```
port ge0
	service-instance ge0
		encapsulation untagged
		connect ip interface ISP
		ctrl+d
	ctrl+d
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```

### BR-RTR 
Переходим к режиму администрирования
```
en
```

Переходим в конфигурационный режим
```
config
```

Задаём имя для роутера
```
hostname BR-RTR.au-team.irpo
```

Создаем интерфейсы и присоединяем к портам
```
interface ISP  
	ip nat outside
	ip address 172.16.2.2/28
	no shutdown
	ctrl+d
port ge0
	service-instance ge0
		encapsulation untagged
			connect ip interface ISP
			ctrl+d
		ctrl+d
interface LAN
	ip nat inside
	ip mtu 1500
	ip address 192.168.3.1/28
	no shutdown
	ctrl+d
port te0
	mtu 9234
	service-instance te0
		encapsulation untagged
		connect ip interface LAN
		ctrl+d
	ctrl+d
```

Cохраняем конфигурацию
```
ctrl+z
write memory
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
Создаём файл и записываем в него
```bash
nano /etc/net/ifaces/ens33.100/options 
```

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
service network restart
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

В файле изменяем /etc/net/ifaces/ens33/options следующие 
```bash
nano /etc/net/ifaces/ens33/options
```

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
service network restart
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

Создаём папку для под-адаптера
```bash
mkdir /etc/net/ifaces/ens33.200
```

Создаём файл и записываем в него
```bash
nano /etc/net/ifaces/ens33.200/options
```

```bash
TYPE=vlan
HOST=ens33
VID=200
BOOTPROTO=dhcp
NM_CONTROLLED=no 
DISABLED=no
```

Cохраняем (ctrl+x, y, enter)

Перезапускаем службу отвечающую за сеть
```bash
service network restart
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
service network restart
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
config
username net_admin
	password P@ssw0rd
	role admin
	ctrl+d
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```
### HQ-SRV и BR-SRV
```bash
useradd -m sshuser -u 2026 
passwd sshuser 
P@ssw0rd #Вводим пароль
P@ssw0rd #Вводим его второй раз для подтверждения
```

В файле /etc/sudoers убираем комментарий у строчки WHEEL_USERS ALL=(ALL:ALL) NOPASSWD: ALL
```bash
nano /etc/sudoers
```

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
port te0 
	mtu 9234
	service-instance te0/vlan100
		encapsulation dot1q 100
		rewrite pop 1
		connect ip interface SRV
		ctrl+d
	service-instance te0/vlan200
		encapsulation dot1q 200
		rewrite pop 1
		connect ip interface CLI
		ctrl+d
	service-instance te0/vlan999
		encapsulation dot1q 999
		rewrite pop 1
		connect ip interface MGMT
		ctrl+d
	ctrl+d
```

Cохраняем конфигурацию
```
crtl+z
write memory
```
## 5. Настройте безопасный удаленный доступ на серверах HQ-SRV и BR-SRV
* Для подключения используйте порт 2026
* Разрешите подключения исключительно пользователю sshuser
* Ограничьте количество попыток входа до двух 
* Настройте баннер «Authorized access only».

Перед началом убедимся, что пакет openssh-sever установлен
```bash
apt-get update
apt-get install openssh-server
```

Редактируем /etc/openssh/sshd_config
```bash
nano /etc/openssh/sshd_config
```

Изменяем следующие параметры
```bash
Port 2026
MaxAuthTries 2
Banner /etc/openssh/banner
PermitRootLogin no
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
config
interface tunnel.1 
	ip mtu 1400
	ip address 10.10.10.1/30
	ip tunnel 172.16.1.2 172.16.2.2 mode gre
	ctrl+d
ip route 0.0.0.0/0 172.16.1.1
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```
### BR-RTR

```
config
interface tunnel.1
	ip mtu 1400
	ip address 10.10.10.2/30
	ip tunnel 172.16.2.2 172.16.1.2 mode gre
	ctrl+d
ip route 0.0.0.0/0 172.16.2.1
```

Cохраняем конфигурацию
```
crtl+z
write memory
```

## 7. Обеспечьте динамическую маршрутизацию на маршрутизаторах HQ-RTR и BR-RTR
* Сети одного офиса должны быть доступны из другого офиса и наоборот. Для обеспечения динамической маршрутизации используйте link state протокол на усмотрение участника
* Разрешите выбранный протокол только на интерфейсах ip туннеля
* Маршрутизаторы должны делиться маршрутами только друг с другом
* Обеспечьте защиту выбранного протокола посредством парольной защиты
* Сведения о настройке и защите протокола занесите в отчёт.
### HQ-RTR
```
config
router ospf 1
	network 10.10.10.0 0.0.0.3 area 0.0.0.0
	network 192.168.1.0 0.0.0.31 area 0.0.0.0
	network 192.168.2.0 0.0.0.15 area 0.0.0.0
	ctrl+d
interface tunnel.1
	ip ospf authentication message-digest
	ip ospf message-digest-key 1 md5 P@ssw0rd
	ip ospf network point-to-point
	ctrl+d
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```

### BR-RTR
```
config
router ospf 1
	network 10.10.10.0 0.0.0.3 area 0.0.0.0
	network 192.168.3.0 0.0.0.15 area 0.0.0.0
	ctrl+d
interface tunnel.1
	ip ospf authentication message-digest
	ip ospf message-digest-key 1 md5 P@ssw0rd
	ip ospf network point-to-point
	ctrl+d
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```

## 8. Настройка динамической трансляции адресов маршрутизаторах HQ-RTR и BR-RTR
* Настройте динамическую трансляцию адресов для обоих офисов в сторону ISP, все устройства в офисах должны иметь доступ к сети Интернет

### HQ-RTR

```
config
ip nat pool NAT 192.168.1.1-192.168.1.30,192.168.2.1-192.168.2.14
ip nat source dynamic inside-to-outside pool NAT overload interface ISP
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```

### BR-RTR

```
config
ip nat pool NAT 192.168.3.1-192.168.3.14
ip nat source dynamic inside-to-outside pool NAT overload interface ISP
```

Cохраняем конфигурацию
```
ctrl+z
write memory
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
Переходим в конфигурационный режим
```
config
```

Создаём пул адресов
```
ip pool Vlan200 192.168.2.1-192.168.2.14
```

Настраиваем dhcp-сервер
```
dhcp-server 1 
	lease 86400
	pool Vlan200 1
		dns 192.168.1.30
		domain-name au-team.irpo
		gateway 192.168.2.1
		mask 28
		ctrl+d
	ctrl+d
```

Выбираем интерфейс на котором будет работать dhcp-сервер
```
interface CLI
	dhcp-server 1
	ctrl+d
```

Cохраняем конфигурацию
```
ctrl+z
write memory
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
| HQ-CLI                                        | hq-cli.au-team.irpo | A, PTR | 192.168.2.2<br>(Смотрите по DHCP) |
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

Обновляем список репозиториев и устанавливаем пакет dnsmasq
```bash
apt-get update
apt-get install dnsmasq
```

Редактируем конфигурационный файл dnsmasq.conf
```bash
nano /etc/dnsmasq.conf
```

И добавляем в неё строки (для удобства прям с первой строки файла):
```bash
no-resolv
domain=au-team.irpo
server=77.88.8.8
interface=ens33.100
                                      
address=/hq-rtr.au-team.irpo/192.168.1.1
ptr-record=1.1.168.192.in-addr.arpa,hq-rtr.au-team.irpo

address=/br-rtr.au-team.irpo/192.168.2.1

address=/hq-srv.au-team.irpo/192.168.1.30
ptr-record=30.1.168.192.in-addr.arpa,hq-srv.au-team.irpo

address=/hq-cli.au-team.irpo/192.168.2.2
ptr-record=2.2.168.192.in-addr.arpa,hq-cli.au-team.irpo

address=/br-srv.au-team.irpo/192.168.3.14

address=/docker.au-team.irpo/172.16.1.1

address=/web.au-team.irpo/172.16.2.1
```

Сохраняем и выходим из файла (ctrl+x, y enter)

Добавляем адрес сервера в файл hosts
```bash
nano /etc/hosts
```

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

Сохраняем и выходим из файла (ctrl+x, y enter)
## 11. Настройте часовой пояс на всех устройствах 
* за исключением виртуального коммутатора, в случае его использования
* согласно месту проведения экзамена

### HQ-RTR, BR-RTR
```
config
ntp timezone utc+8
```

Cохраняем конфигурацию
```
ctrl+z
write memory
```
### ISP, HQ-SRV, HQ-CLI, BR-SRV
Выключаем службу chrony:
```bash
systemctl disable --now chronyd
```

Обновляем список пакетов и скачиваем службу systemd-timesyncd:
```bash
apt-get update
apt-get install systemd-timesyncd
```

Теперь включим службу systemd-timesyncd и посмотрим её статус работы:
```bash
systemctl enable --now systemd-timesyncd
timedatectl set-timezone Asia/Irkutsk
timedatectl timesync-status
```

