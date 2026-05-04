## Цель работы
Научиться конфигурировать и тестировать службу имён для корпоративной сети на основе DNS-сервера BIND.

## Стек технологий

- **ОС:** Linux (Ubuntu)
- **DNS-сервер:** BIND9 (`bind9`)
- **Утилиты:** `nslookup`, `dig`, `systemctl`

## Ход выполнения работы

### Часть 1. Создание прямой и обратной зон

#### 1.1 Установка BIND9

Установка пакета `bind9` выполнена командой:
```bash
sudo apt-get update
sudo apt-get install bind9 -y
```

#### 1.2 Конфигурация параметров сервера

Отредактирован файл /etc/bind/named.conf.options:
```bash
options {
    directory "/var/cache/bind";
    forwarders {
        8.8.8.8;
    };
    listen-on {
        172.16.X.0;  # X — номер машины в классе
    };
    allow-query {
        any;
    };
};
```

Отредактирован файл /etc/bind/named.conf.local:
```bash
zone "arg.local" {
    type master;
    file "/etc/bind/db.arg.local";
};

zone "0.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16.0";
};
```

### 1.3 Создание файлов зон

Файлы зон созданы копированием шаблона db.local:
```bash
sudo cp /etc/bind/db.local /etc/bind/db.arg.local
sudo cp /etc/bind/db.local /etc/bind/db.172.16.0
```

Файл прямой зоны /etc/bind/db.arg.local:
```bash
$TTL    604800
@       IN      SOA     arg.local. admin.arg.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.arg.local.
@       IN      A       172.16.X.1
ns      IN      A       172.16.X.1
www     IN      A       172.16.X.10
mail    IN      A       172.16.X.20
```

Файл обратной зоны /etc/bind/db.172.16.0:
```bash
$TTL    604800
@       IN      SOA     arg.local. admin.arg.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.arg.local.
1       IN      PTR     ns.arg.local.
10      IN      PTR     www.arg.local.
20      IN      PTR     mail.arg.local.
```

### 1.4 Проверка конфигурации

Проверка файлов зон выполнена командой:
```bash
sudo named-checkzone arg.local /etc/bind/db.arg.local
sudo named-checkzone 0.16.172.in-addr.arpa /etc/bind/db.172.16.0
```

Перезапуск сервера:
```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

### 1.5 Тестирование с помощью nslookup
```bash
nslookup www.arg.local
nslookup 172.16.X.10
```

### Часть 2. Тестирование зон с помощью dig

### Проверка прямой зоны:
```bash
dig www.arg.local
```

Проверка обратной зоны:
```bash
dig -x 172.16.X.10
```

Команда dig отображает:

- QUESTION SECTION — отправленный запрос
- ANSWER SECTION — полученный ответ
- статистику запроса

### Часть 3. Разрешение внешних имён

Проверка разрешения внешних имён через настроенный DNS-сервер:
```bash
dig slashdot.org
dig github.com
```

Сервер успешно разрешил внешние имена через forwarders.

### Часть 4. Настройка вторичного DNS-сервера

### 4.1 Настройка первичного сервера (Master)

В файл /etc/bind/named.conf.local добавлены разрешения на передачу зоны:
```bash
zone "arg.local" {
    type master;
    file "/etc/bind/db.arg.local";
    allow-transfer { 172.16.X.2; };
    also-notify { 172.16.X.2; };
};

zone "0.16.172.in-addr.arpa" {
    type master;
    file "/etc/bind/db.172.16.0";
    allow-transfer { 172.16.X.2; };
    also-notify { 172.16.X.2; };
};
```

В файл /etc/bind/named.conf.options добавлен ACL trusted:
```bash
acl trusted {
    172.16.X.1;
    172.16.X.2;
};

options {
    allow-query { trusted; };
    allow-transfer { trusted; };
};
```

### 4.2 Настройка вторичного сервера (Slave)

Файл /etc/bind/named.conf.options:
```bash
options {
    directory "/var/cache/bind";
    forwarders {
        8.8.8.8;
    };
    listen-on {
        172.16.X.2;
    };
    allow-query {
        any;
    };
};
```

Файл /etc/bind/named.conf.local:
```bash
zone "arg.local" {
    type slave;
    file "/var/cache/bind/db.arg.local";
    masters { 172.16.X.1; };
};

zone "0.16.172.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.172.16.0";
    masters { 172.16.X.1; };
};
```

### 4.3 Проверка работы вторичного сервера

Проверка через nslookup с указанием вторичного сервера:
```bash
nslookup www.arg.local 172.16.X.2
```

Проверка через dig с указанием вторичного сервера:
```bash
dig @172.16.X.2 www.arg.local
```

### Результаты:

| Тип проверки |	Команда	| Результат |
|:---|:---:|---:|
| Прямая зона |	nslookup www.arg.local |	Получен IP-адрес 172.16.X.10 |
| Обратная зона |	nslookup 172.16.X.10 |	Получено имя www.arg.local |
| Внешнее имя |	dig github.com |	Получен A-запись |
| Вторичный сервер |	dig @172.16.X.2 www.arg.local |	Получен корректный ответ |


### Вывод

В ходе выполнения лабораторной работы:
- Установлен и настроен DNS-сервер BIND9
- Созданы прямая и обратная зоны для приватной сети
- Выполнено тестирование зон с помощью nslookup и dig
- Настроено разрешение внешних имён через forwarders
- Сконфигурирован вторичный DNS-сервер и проверена его работа
- Получены навыки администрирования DNS в корпоративной среде
