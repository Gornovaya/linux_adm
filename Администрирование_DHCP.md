## Стек технологий
- **Виртуализация:** VirtualBox
- **ОС:** Linux (Ubuntu)
- **Сервис:** ISC DHCP Server (`isc-dhcp-server`)
- **Утилиты:** `ip`, `ifconfig`, `netplan`, `systemctl`

## Ход выполнения работы

### 1. Установка DHCP-сервера

Установка пакета `isc-dhcp-server` выполнена командой:

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

Проверка установки:
```bash
dpkg -l | grep isc-dhcp-server
```

### 2. Настройка сети в VirtualBox

В настройках виртуальной машины тип подключения изменён на Сетевой мост (Bridge).

Определение имени сетевого интерфейса выполнено командой:
```bash
ip addr
```

В результате определён активный интерфейс: enp0s3.

### 3. Настройка DHCP-сервера

3.1 Редактирование /etc/default/isc-dhcp-server

В файле указан интерфейс, на котором сервер будет принимать запросы:
```bash
INTERFACESv4="enp0s3"
```

3.2 Редактирование /etc/dhcp/dhcpd.conf

Основной файл конфигурации настроен следующим образом:
```bash
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.100 192.168.1.200;
    option routers 192.168.1.1;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    default-lease-time 600;
    max-lease-time 7200;
}
```

Пояснение параметров:
- subnet — определение подсети
- range — диапазон динамически выдаваемых IP-адресов
- option routers — шлюз по умолчанию (указывается IP роутера в локальной сети)
- option domain-name-servers — DNS-серверы
- default-lease-time — стандартное время аренды адреса (сек)
- max-lease-time — максимальное время аренды адреса (сек)


3.3 Запуск сервера
Запуск и проверка статуса сервера:
```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

Статус сервера — active (running).

### 4. Настройка клиента для получения IP по DHCP


На клиентской машине отредактирован файл /etc/netplan/01-network-manager-all.yaml:
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: yes
```

Применение настроек:
```bash
sudo netplan apply
```

После применения клиент успешно получил IP-адрес из диапазона, заданного на сервере (192.168.1.100 – 192.168.1.200).

Проверка на клиенте:
```bash
ifconfig
```

Проверка на сервере (файл аренды):
```bash
cat /var/lib/dhcp/dhcpd.leases
```


### 5. Назначение статического IP по MAC-адресу

В файл /etc/dhcp/dhcpd.conf добавлена секция host:
```bash
host client {
    hardware ethernet 08:00:27:ab:cd:ef;
    fixed-address 192.168.1.210;
}
```

Перезапуск сервера:
```bash
sudo systemctl restart isc-dhcp-server
```

На клиенте обновлена аренда:
```bash
sudo netplan try
```

После обновления клиент получил статически закреплённый IP-адрес 192.168.1.210.

### Результаты.
| Тип выдачи |	Результат |
| :--- | :--- | 
| Динамическая |	Клиент получил IP-адрес из диапазона 192.168.1.100 – 192.168.1.200 |
| Статическая	| Клиент с MAC-адресом 08:00:27:ab:cd:ef получил фиксированный адрес 192.168.1.210 |
