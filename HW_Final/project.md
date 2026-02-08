## 1. Подготовка и описание стенда

**Для тестового стенда выбрана конфигурация из трех виртуальных машин.**
- 3 ВМ для etcd
- 3 ВМ для patroni
- 3 ВМ для postgresql

| NAME      | INTERNAL IP   |
|-----------|---------------|
| pg-srv01  | 192.168.0.10  |
| pg-srv02  | 192.168.0.11  |
| pg-srv03  | 192.168.0.12  |



**Архитектура кластера:**
```
[PostgreSQL Master] [PostgreSQL Replica] [PostgreSQL Replica]
     Patroni           Patroni             Patroni
      etcd              etcd                 etcd
```
Для развертывания ВМ будем использовать Yandex.Cloud

## 2. Конфигуригурация Яндекс.Облако

**Подключаемся и выполняем конфигурацию окружения с помощью команды:**
```bash
yc init
```
На этом этапе ответим "y" и укажем дефолтную Compute zone, чтобы не получить ошибки в командах в дальнейшем (в них не указываем Compute zone)

```console
Do you want to configure a default Compute zone? [Y/n] y
Which zone do you want to use as a profile default?
 [1] ru-central1-a
 [2] ru-central1-b
 [3] ru-central1-d
 [4] ru-central1-k
 [5] Don't set default zone
Please enter your numeric choice: 1
Your profile default Compute zone has been set to 'ru-central1-a'
```

**Создаём сеть:**
```bash
yc vpc network create --name otus-pg-net --description "otus-pg-net"
```
```console
id: enplhbro9rlnibb7tgrb
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-04T08:47:47Z"
name: otus-pg-net
description: otus-pg-net
default_security_group_id: enp10pb9p3outvci588d
```

**Создаём подсеть:**
```bash
yc vpc subnet create --name otus-pg-subnet --range 192.168.0.0/24 --network-name otus-pg-net --description "otus-pg-subnet"
```
```console
id: e9b57fk4u523knrmm4e7
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-04T08:48:38Z"
name: otus-pg-subnet
description: otus-pg-subnet
network_id: enplhbro9rlnibb7tgrb
zone_id: ru-central1-a
v4_cidr_blocks:
  - 192.168.0.0/24
```

### 2.1 Нарезаем виртуальные машинки (ВМ).

В качестве операционной системы на всех ВМ используем Ubuntu 24.04 lts.
Вычислительные ресурсы заказываем минимально необходимые,  т.к. практикуемся в установке и настройке, производительность не планируется оценивать.

**ВМ 1:**
```bash
yc compute instance create `
    --name pg-srv01 `
    --hostname pg-srv01 `
    --cores 2 `
    --memory 4 `
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts `
    --network-interface subnet-name=otus-pg-subnet,nat-ip-version=ipv4,ipv4-address=192.168.0.10 `
    --ssh-key yc_key.pub
```
```console
done (29s)
id: fhmhtdu7e8shem1p4ljh
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-08T10:34:00Z"
name: pg-srv01
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmp81usdgq6mn94t9us
  auto_delete: true
  disk_id: fhmp81usdgq6mn94t9us
network_interfaces:
  - index: "0"
    mac_address: d0:0d:11:eb:7c:77
    subnet_id: e9b57fk4u523knrmm4e7
    primary_v4_address:
      address: 192.168.0.10
      one_to_one_nat:
        address: 93.77.182.160
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: pg-srv01.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V2
application: {}
```

**ВМ 2:**
```bash
yc compute instance create `
    --name pg-srv02 `
    --hostname pg-srv02 `
    --cores 2 `
    --memory 4 `
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts `
    --network-interface subnet-name=otus-pg-subnet,nat-ip-version=ipv4,ipv4-address=192.168.0.11 `
    --ssh-key yc_key.pub
```
```console
done (36s)
id: fhmt1j8apbd4q0o5dhkf
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-08T10:34:55Z"
name: pg-srv02
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhmes8eck4jr6bt56s6e
  auto_delete: true
  disk_id: fhmes8eck4jr6bt56s6e
network_interfaces:
  - index: "0"
    mac_address: d0:0d:1d:0c:d0:ac
    subnet_id: e9b57fk4u523knrmm4e7
    primary_v4_address:
      address: 192.168.0.11
      one_to_one_nat:
        address: 93.77.180.139
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: pg-srv02.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V2
application: {}
```

**ВМ 3:**
```bash
yc compute instance create `
    --name pg-srv03 `
    --hostname pg-srv03 `
    --cores 2 `
    --memory 4 `
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts `
    --network-interface subnet-name=otus-pg-subnet,nat-ip-version=ipv4,ipv4-address=192.168.0.12 `
    --ssh-key yc_key.pub
```
```console
done (28s)
id: fhmocu4atihh00jgbk06
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-08T10:35:48Z"
name: pg-srv03
zone_id: ru-central1-a
platform_id: standard-v2
resources:
  memory: "4294967296"
  cores: "2"
  core_fraction: "100"
status: RUNNING
metadata_options:
  gce_http_endpoint: ENABLED
  aws_v1_http_endpoint: ENABLED
  gce_http_token: ENABLED
  aws_v1_http_token: DISABLED
boot_disk:
  mode: READ_WRITE
  device_name: fhm1jg9o1e6s1kqomdam
  auto_delete: true
  disk_id: fhm1jg9o1e6s1kqomdam
network_interfaces:
  - index: "0"
    mac_address: d0:0d:18:67:88:ae
    subnet_id: e9b57fk4u523knrmm4e7
    primary_v4_address:
      address: 192.168.0.12
      one_to_one_nat:
        address: 93.77.180.108
        ip_version: IPV4
serial_port_settings:
  ssh_authorization: OS_LOGIN
gpu_settings: {}
fqdn: pg-srv03.ru-central1.internal
scheduling_policy: {}
network_settings:
  type: STANDARD
placement_policy: {}
hardware_generation:
  legacy_features:
    pci_topology: PCI_TOPOLOGY_V2
application: {}
```

**Cписок готовых инстансов:**
```bash
yc compute instance list
```
```console
+----------------------+----------+---------------+---------+---------------+--------------+
|          ID          |   NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP  |
+----------------------+----------+---------------+---------+---------------+--------------+
| fhmhtdu7e8shem1p4ljh | pg-srv01 | ru-central1-a | RUNNING | 93.77.182.160 | 192.168.0.10 |
| fhmocu4atihh00jgbk06 | pg-srv03 | ru-central1-a | RUNNING | 93.77.180.108 | 192.168.0.12 |
| fhmt1j8apbd4q0o5dhkf | pg-srv02 | ru-central1-a | RUNNING | 93.77.180.139 | 192.168.0.11 |
+----------------------+----------+---------------+---------+---------------+--------------+
```

**Подключаемся к ВМ:**
```bash
ssh -i ~/.ssh/yc_key yc-user@93.77.182.160
ssh -i ~/.ssh/yc_key yc-user@93.77.180.139
ssh -i ~/.ssh/yc_key yc-user@93.77.180.108
```

**Устанавливаем обновления и базовый набор инструментов на все машины:**
```bash
sudo apt update &&
sudo apt upgrade -y &&
sudo apt install -y curl wget zip unzip tar nano htop &&
sudo apt autoremove -y
```

На этом подготовка тестового стенда завершена.


## 3. Установка и настройка ETCD

**Ставим и настраиваем etcd на всех трёх нодах:**
```bash
sudo apt -y install etcd-server
sudo apt -y install etcd-client
```
**Останавливаем etcd:**
```bash
sudo systemctl stop etcd
sudo systemctl disable etcd
```
**Удаляем дефолтный конфиг etcd:**
```bash
sudo rm -rf /var/lib/etcd/default
```

**Создаём конфиг на хостах etcd, для каждого хоста свой конфиг:**
```bash
sudo vi /etc/default/etcd
```

- Конфиг на pg-srv01: [`/etc/default/etcd`](config/etcd-1_config)
- Конфиг на pg-srv02: [`/etc/default/etcd`](config/etcd-2_config)
- Конфиг на pg-srv03: [`/etc/default/etcd`](config/etcd-3_config)

**После изменения конфигов, выполняем на всех хостах:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
```

**Запускаем кластер, начиная с первой ноды:**
```bash
sudo systemctl start etcd
```
**Проверим, слушает ли etcd порт:**
```bash
sudo ss -tlnp | grep 2380
```
```console
LISTEN 0      4096    192.168.0.10:2380      0.0.0.0:*    users:(("etcd",pid=8879,fd=7))
```
**После запуска, проверяем состав и состояние кластера. На любой ноде:**
```bash
etcdctl endpoint status --cluster -w table
```
```console
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://192.168.0.12:2379 |  abcdcfa913092d3 |  3.4.30 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
| http://192.168.0.11:2379 | 55de3230f72f63bd |  3.4.30 |   20 kB |     false |      false |         2 |          9 |                  9 |        |
| http://192.168.0.10:2379 | f31b755061547989 |  3.4.30 |   20 kB |      true |      false |         2 |          9 |                  9 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
Кластер etcd работает, лидер выбран (pg-srv01), ошибок нет.
Важно отметить, какой хост выбран лидером. Обычно это первый запущенный в кластере хост.
Настройку patroni рекомендуется начинать на хосте, который будет лидером.

## 4. Установка и настройка PostgreSQL

### 4.1 Устанавливаем postgres 18 на всех трех хостах:
```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt-get -y install postgresql
```
**Проверяем версию и состояние кластера postgres на каждом хосте:**
```bash
pg_lsclusters
sudo -u postgres psql -c "SELECT version();"
```
```console
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
-----------------------------------------------------------------------------------------------------------------------------------
 PostgreSQL 18.1 (Ubuntu 18.1-1.pgdg24.04+2) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
(1 row)
```
**Проверить состояние кластера:**
```bash
pg_lsclusters
```
```console
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```
**Меняем пароль пользователя postgres на всех хостах:**
```bash
sudo -u postgres psql -c "alter user postgres with ENCRYPTED password 'Oracle4U'"
```

### 4.2 Настройка postgres на мастер ноде
**Только на хосте pg-srv01 создаём БД и пользователя replicator:**
```bash
sudo -u postgres psql -c "create database otus"
sudo -u postgres psql -d otus -c "create user replicator replication login ENCRYPTED password 'Oracle4U'"
sudo -u postgres psql -d otus -c "create table test(c1 text)"
sudo -u postgres psql -d otus -c "insert into test values('test1')"
sudo -u postgres psql -d otus -c "insert into test values('test2')"
sudo -u postgres psql -d otus -c "insert into test values('test3')"
sudo -u postgres psql -d otus -c "select * from test"
```

**Редактируем /etc/postgresql/18/main/pg_hba.conf на pg-srv01 новым содержимым:**
```bash
  cat << 'EOF' | sudo tee /etc/postgresql/18/main/pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local socket connections
local   all             postgres                                peer
local   all             all                                     scram-sha-256

# TCP/IP connections
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             192.168.0.10/32         scram-sha-256
host    all             all             192.168.0.0/24          scram-sha-256
host    replication     replicator      192.168.0.0/24          scram-sha-256
EOF
```

**На ноде pg-srv01 редактируем /etc/postgresql/18/main/postgresql.conf командой, добавим listen_addresses = '*' в конец файла:**
```bash
echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/18/main/postgresql.conf
```

**Перезапускаем postgres на pg-srv-01:**
```bash
sudo systemctl restart postgresql
```
Проверяем состояние кластера после старта:
```bash
pg_lsclusters
```
Кластер запущен:
```console
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

**Проверяем, какие порты слушает postgres после изменений конфигурации:**
```bash
sudo ss -tlnp | grep 5432
```
```console
LISTEN 0      200          0.0.0.0:5432      0.0.0.0:*    users:(("postgres",pid=12017,fd=6))
LISTEN 0      200             [::]:5432         [::]:*    users:(("postgres",pid=12017,fd=7))
```
Postgres cлушает порт 5432 со всех адресов. То что нужно
Настройка первой ноды-мастера завершена.

### 4.2 Настройка postgres на хостах для реплик

**На нодах pg-srv02 и pg-srv03 удаляем каталог /var/lib/postgresql/18/main и создаём заново:**
```bash
sudo systemctl stop postgresql
sudo pkill -9 postgres
sudo rm -rf /var/lib/postgresql/18/main
sudo mkdir -p /var/lib/postgresql/18/main
sudo chown postgres:postgres /var/lib/postgresql/18/main
sudo chmod 700 /var/lib/postgresql/18/main
sudo ls -la /var/lib/postgresql/18/main
```

**Текущий статус:**
- Кластер etcd работает на 3-х нодах.
- На pg-srv01 установлена, сконфигурирована для подключений postgresql, запущена и слушает порт 5432 со всех адресов.
- На pg-srv02 и pg-srv03 postgres остановлена, каталог /var/lib/postgresql/18/main очищен. После старта кластера patrini, данные и конфигурационные паараметры должны быть реплицированы с мастер ноды.

## 5. Установка и настройка Patroni

### 5.1 Установка и конфигурирование patroni на каждой ноде.

Новый механизмом безопасности в Ubuntu 22.04+ (и Debian 12+) блокирует глобальную установку
Python-пакетов через pip, чтобы не повредить системные пакеты.

- Ставим модуль для создания виртуальных окружений:
```bash
sudo apt install python3.12-venv
```
- Создаём каталог для Patroni:
```bash
sudo mkdir -p /opt/patroni
```
- Передаём владение каталогом пользователю postgres:
```bash
sudo chown postgres:postgres /opt/patroni
```
- Создаём виртуальное окружение от имени postgres:
```bash
sudo -u postgres python3 -m venv /opt/patroni/venv
```
- Устанавливаем Patroni с поддержкой etcd3:
```bash
sudo -u postgres /opt/patroni/venv/bin/pip install 'patroni[etcd3]'
```
- Устанавливаем драйвер для работы Patroni с PostgreSQL:
```bash
sudo -u postgres /opt/patroni/venv/bin/pip install 'psycopg2-binary'
```

**Создадим конфигурационный файл patroni, для каждой ноды свой:**
```bash
sudo mkdir -p /etc/patroni
sudo vi /etc/patroni/patroni.yml
```
- Конфиг patroni на pg-srv01: [`/etc/patroni/patroni.yml`](config\srv-pg01_patroni.yml)
- Конфиг patroni на pg-srv02: [`/etc/patroni/patroni.yml`](config\srv-pg01_patroni.yml)
- Конфиг patroni на pg-srv03: [`/etc/patroni/patroni.yml`](config\srv-pg01_patroni.yml)

**Дадим права на конфиг:**
```bash
sudo chown postgres:postgres /etc/patroni/patroni.yml
sudo chmod 600 /etc/patroni/patroni.yml
```

**Определяем patroni как службу:**
```bash
sudo tee /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
Environment=PATH=/opt/patroni/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ExecStart=/opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```
**Перезагружаем службу:**
```bash
sudo systemctl daemon-reload
```

**Проверяем что всё установилось, но кластер не инициализирован:**
```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (uninitialized) ------------+-----+------------+-----+
| Member | Host | Role | State | TL | Receive LSN | Lag | Replay LSN | Lag |
+--------+------+------+-------+----+-------------+-----+------------+-----+
+--------+------+------+-------+----+-------------+-----+------------+-----+
```

### 5.2 Запуск кластера patroni на мастер ноде

**Запускаем patroni, вначале на первой ноде:**
```bash
sudo systemctl enable patroni
sudo systemctl start patroni
sudo systemctl status patroni
```

**Смотрим состояние кластера:**
```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) -+----+-------------+-----+------------+-----+
| Member   | Host         | Role   | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+--------+---------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader | running |  2 |             |     |            |     |
+----------+--------------+--------+---------+----+-------------+-----+------------+-----+
```

Кластер запустился, нода pg-srv01 с ролью лидера. Можно стартовать на оставшихся нодах.

❗️Если этом этапе, если возникли проблемы - лучше и проще решать на первой ноде и даже не пытаться запускать patroni на остальных нодах.

Patroni успешно запускает кластер в составе одной ноды, выбирает лидера и готов к подключению реплик. 

**Анализ возможных проблем приведен в последнем разделе документа.**

### 5.2 Запуск patroni на второй и третьей нодах

В предыдущем разделе установили patroni на второй и треьей ноде, добавили конфигурационный файл.
Если этого не сделали - можно сделать сейчас.

**Запускаем patroni последовательно, вначале на второй, потом на треьей ноде**
```bash
sudo systemctl enable patroni
sudo systemctl start patroni
sudo systemctl status patroni
```

Проверяем статус patroni на pg-srv02 и pg-srv03 (вывод должен быть одинаковый):
```bash
sudo systemctl status patroni
```
```console
● patroni.service - High availability PostgreSQL Cluster
     Loaded: loaded (/etc/systemd/system/patroni.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-02-08 12:52:15 UTC; 45s ago
   Main PID: 12462 (patroni)
      Tasks: 16 (limit: 4653)
     Memory: 118.6M (peak: 119.1M)
        CPU: 1.308s
     CGroup: /system.slice/patroni.service
             ├─12462 /opt/patroni/venv/bin/python3 /opt/patroni/venv/bin/patroni /etc/patroni/patroni.yml
             ├─12479 /usr/lib/postgresql/18/bin/postgres -D /var/lib/postgresql/18/main --config-file=/var/lib/postgresql/18/main/postgresql.conf --listen_addresses=192.168.0.12 --port=5432 --cluster_name=pg-clus>
             ├─12481 "postgres: pg-cluster: io worker 0"
             ├─12482 "postgres: pg-cluster: io worker 1"
             ├─12483 "postgres: pg-cluster: io worker 2"
             ├─12484 "postgres: pg-cluster: checkpointer "
             ├─12485 "postgres: pg-cluster: background writer "
             ├─12486 "postgres: pg-cluster: startup recovering 000000020000000000000005"
             ├─12495 "postgres: pg-cluster: postgres postgres 192.168.0.12(45688) idle"
             └─12498 "postgres: pg-cluster: walreceiver streaming 0/5000060"

Feb 08 12:52:22 pg-srv03 patroni[12462]: 2026-02-08 12:52:22,378 INFO: Lock owner: pg-srv01; I am pg-srv03
Feb 08 12:52:22 pg-srv03 patroni[12462]: 2026-02-08 12:52:22,469 INFO: bootstrap from leader 'pg-srv01' in progress
Feb 08 12:52:22 pg-srv03 patroni[12492]: 192.168.0.12:5432 - accepting connections
Feb 08 12:52:22 pg-srv03 patroni[12462]: 2026-02-08 12:52:22,591 INFO: Lock owner: pg-srv01; I am pg-srv03
Feb 08 12:52:22 pg-srv03 patroni[12462]: 2026-02-08 12:52:22,591 INFO: establishing a new patroni heartbeat connection to postgres
Feb 08 12:52:22 pg-srv03 patroni[12462]: 2026-02-08 12:52:22,757 INFO: no action. I am (pg-srv03), a secondary, and following a leader (pg-srv01)
Feb 08 12:52:26 pg-srv03 patroni[12498]: 2026-02-08 12:52:26.636 UTC [12498] LOG:  started streaming WAL from primary at 0/5000000 on timeline 2
Feb 08 12:52:32 pg-srv03 patroni[12462]: 2026-02-08 12:52:32,472 INFO: no action. I am (pg-srv03), a secondary, and following a leader (pg-srv01)
Feb 08 12:52:42 pg-srv03 patroni[12462]: 2026-02-08 12:52:42,923 INFO: no action. I am (pg-srv03), a secondary, and following a leader (pg-srv01)
Feb 08 12:52:52 pg-srv03 patroni[12462]: 2026-02-08 12:52:52,923 INFO: no action. I am (pg-srv03), a secondary, and following a leader (pg-srv01)
```

**После запуска patroni на pg-srv02 и pg-srv03, проверим состояние кластера:**
```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   |  2 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming |  2 |   0/5000060 |   0 |  0/5000060 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming |  2 |   0/5000060 |   0 |  0/5000060 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
**Кластер patroni запущен и работает. Нода pg-srv01, ноды pg-srv02 и pg-srv03 реплики. Ошибок нет.**


## 7. Режим switchower

Режим предназначен для ручного переключения мастер ноды. Например, необходимо вывести мастер ноду на обслуживание.

Моделируем switchower.
Прежде чем начинать, проверим состояние базы данных otus на мастер ноде и добавим запись в таблицу.
**Смотрим состояние кластера:**
```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   |  2 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming |  2 |   0/5000168 |   0 |  0/5000168 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming |  2 |   0/5000168 |   0 |  0/5000168 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
Лидер pg-srv01, выполним на нём запросы:
sudo -u postgres psql -d otus -c "select * from test"
sudo -u postgres psql -d otus -c "insert into test values('test10')"
sudo -u postgres psql -d otus -c "insert into test values('test20')"
sudo -u postgres psql -d otus -c "select * from test"
```
```console
INSERT 0 1
INSERT 0 1
   c1
--------
 test1
 test2
 test3
 test10
 test20
(5 rows)
```
А затем выполним запрос на ноде реплике, pg-srv02:
```bash
sudo -u postgres psql -d otus -c "select * from test"
```
```console
   c1
--------
 test1
 test2
 test3
 test10
 test20
(5 rows)
```
Изменения с мастер ноды успешно реплицированы на ноду pg-srv02.
```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   |  2 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming |  2 |   0/5000310 |   0 |  0/5000310 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming |  2 |   0/5000310 |   0 |  0/5000310 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
Значения в Receive LSN и Replay LSN увеличились, реплики приняли и накатили изменения.


Выполним команду переключения лидера:
```bash
sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml switchover
```
```
Current cluster topology
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   |  3 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming |  3 |   0/9000168 |   0 |  0/9000168 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming |  3 |   0/9000168 |   0 |  0/9000168 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
Primary [pg-srv01]:
Candidate ['pg-srv02', 'pg-srv03'] []: pg-srv02
When should the switchover take place (e.g. 2026-02-08T17:27 )  [now]:
Are you sure you want to switchover cluster pg-cluster, demoting current leader pg-srv01? [y/N]:y
2026-02-08 16:27:25.41635 Successfully switched over to "pg-srv02"
+ Cluster: pg-cluster (7604470388804299721) --+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | stopped |    |     unknown |     |    unknown |     |
| pg-srv02 | 192.168.0.11 | Leader  | running |  3 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | running |  3 |   0/90002B0 |   0 |  0/90002B0 |   0 |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
```

Лидер изменился, теперь это pg-srv02. Прежний лидер в состоянии stopped (это временно).
Через 30 секунд запросим состояние кластера:
```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | running   |    |   0/5000000 |   0 |  0/5000560 |   0 |
| pg-srv02 | 192.168.0.11 | Leader  | running   |  3 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming |  3 |   0/50006A0 |   0 |  0/50006A0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```

Зараза! Реплика pg-srv01 не не переходит в состояние streaming... 
На этом этапе решаются проблемы, описанные в разделе "Возможные ошибки". После устранения ошибок, конфликтов и правки конфигов, возвращаемся к проверке режима switchover:

```bash
yc-user@pg-srv01:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   | 12 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming | 12 |   0/F000C38 |   0 |  0/F000C38 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 12 |   0/F000C38 |   0 |  0/F000C38 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
Последовательно переключаем лидера на pg-srv02, pg-srv03 и затем возращаем на pg-srv01. Каждый раз убеждаемся, что ошибок нет и реплики переходят в состояние streaming:
```console
yc-user@pg-srv01:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   | 12 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming | 12 |   0/F000C38 |   0 |  0/F000C38 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 12 |   0/F000C38 |   0 |  0/F000C38 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
Primary [pg-srv01]:
Candidate ['pg-srv02', 'pg-srv03'] []: pg-srv02
When should the switchover take place (e.g. 2026-02-08T21:00 )  [now]:
Are you sure you want to switchover cluster pg-cluster, demoting current leader pg-srv01? [y/N]: y
2026-02-08 20:00:51.81339 Successfully switched over to "pg-srv02"
+ Cluster: pg-cluster (7604470388804299721) --+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | stopped |    |     unknown |     |    unknown |     |
| pg-srv02 | 192.168.0.11 | Leader  | running |    |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | running |    |   0/F000D80 |   0 |  0/F000D80 |   0 |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
yc-user@pg-srv01:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | streaming | 13 |   0/F000D80 |   0 |  0/F000D80 |   0 |
| pg-srv02 | 192.168.0.11 | Leader  | running   | 13 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 13 |   0/F000D80 |   0 |  0/F000D80 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
yc-user@pg-srv01:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | streaming | 13 |   0/F000EC0 |   0 |  0/F000EC0 |   0 |
| pg-srv02 | 192.168.0.11 | Leader  | running   | 13 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 13 |   0/F000EC0 |   0 |  0/F000EC0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
Primary [pg-srv02]:
Candidate ['pg-srv01', 'pg-srv03'] []: pg-srv03
When should the switchover take place (e.g. 2026-02-08T21:01 )  [now]:
Are you sure you want to switchover cluster pg-cluster, demoting current leader pg-srv02? [y/N]: y
2026-02-08 20:01:16.56702 Successfully switched over to "pg-srv03"
+ Cluster: pg-cluster (7604470388804299721) --+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | running |    |   0/F001008 |   0 |  0/F001008 |   0 |
| pg-srv02 | 192.168.0.11 | Replica | stopped |    |     unknown |     |    unknown |     |
| pg-srv03 | 192.168.0.12 | Leader  | running |    |             |     |            |     |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
yc-user@pg-srv01:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | streaming | 14 |   0/F001008 |   0 |  0/F001008 |   0 |
| pg-srv02 | 192.168.0.11 | Replica | streaming | 14 |   0/F001008 |   0 |  0/F001008 |   0 |
| pg-srv03 | 192.168.0.12 | Leader  | running   | 14 |             |     |            |     |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
yc-user@pg-srv01:~$ sudo /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | streaming | 14 |   0/F001148 |   0 |  0/F001148 |   0 |
| pg-srv02 | 192.168.0.11 | Replica | streaming | 14 |   0/F001148 |   0 |  0/F001148 |   0 |
| pg-srv03 | 192.168.0.12 | Leader  | running   | 14 |             |     |            |     |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
Primary [pg-srv03]:
Candidate ['pg-srv01', 'pg-srv02'] []: pg-srv01
When should the switchover take place (e.g. 2026-02-08T21:01 )  [now]:
Are you sure you want to switchover cluster pg-cluster, demoting current leader pg-srv03? [y/N]: y
2026-02-08 20:01:40.28815 Successfully switched over to "pg-srv01"
+ Cluster: pg-cluster (7604470388804299721) --+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State   | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running |    |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | running |    |   0/F001290 |   0 |  0/F001290 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | stopped |    |     unknown |     |    unknown |     |
+----------+--------------+---------+---------+----+-------------+-----+------------+-----+
yc-user@pg-srv01:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   | 15 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming | 15 |   0/F001290 |   0 |  0/F001290 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 15 |   0/F001290 |   0 |  0/F001290 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```

**Испытания режима switchower успешно проведены.**

## 8. Проверка failover

Проверяем, как поведёт себя кластер в случае падения мастера (лидера). Ожидаемое поведение - через некоторое время кластер должен выбрать нового лидера, а при восстановлении работы упавшей ноды, которая ранее была лидером, вернуть её в состав кластера в роли реплики.

На второй ноде смотрим состояние кластера и лидера:
```console
yc-user@pg-srv02:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Leader  | running   | 15 |             |     |            |     |
| pg-srv02 | 192.168.0.11 | Replica | streaming | 15 |   0/F0013D0 |   0 |  0/F0013D0 |   0 |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 15 |   0/F0013D0 |   0 |  0/F0013D0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
Заходим на машину лидера и выполняем выключение:
```bash
yc-user@pg-srv01:~$ sudo poweroff
Broadcast message from root@pg-srv01 on pts/2 (Sun 2026-02-08 20:14:28 UTC):
The system will power off now!
yc-user@pg-srv01:~$ Connection to 93.77.182.160 closed by remote host.
Connection to 93.77.182.160 closed.
```
Возвращаемся на вторую ноду и мониторим состояние кластера:

```console
yc-user@pg-srv02:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | stopped   |    |     unknown |     |    unknown |     |
| pg-srv02 | 192.168.0.11 | Leader  | running   | 16 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 16 |   0/F001588 |   0 |  0/F001588 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
yc-user@pg-srv02:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
2026-02-08 20:14:55,095 - ERROR - Failed to get list of machines from http://192.168.0.10:2379/v3beta: MaxRetryError('HTTPConnectionPool(host=\'192.168.0.10\', port=2379): Max retries exceeded with url: /version (Caused by NewConnectionError("HTTPConnection(host=\'192.168.0.10\', port=2379): Failed to establish a new connection: [Errno 111] Connection refused"))')
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | stopped   |    |     unknown |     |    unknown |     |
| pg-srv02 | 192.168.0.11 | Leader  | running   | 16 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 16 |   0/F0015C0 |   0 |  0/F0015C0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
yc-user@pg-srv02:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
2026-02-08 20:15:44,316 - ERROR - Request to server http://192.168.0.10:2379 failed: MaxRetryError('HTTPConnectionPool(host=\'192.168.0.10\', port=2379): Max retries exceeded with url: /v3/kv/range (Caused by NewConnectionError("HTTPConnection(host=\'192.168.0.10\', port=2379): Failed to establish a new connection: [Errno 111] Connection refused"))')
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv02 | 192.168.0.11 | Leader  | running   | 16 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 16 |   0/F0015C0 |   0 |  0/F0015C0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
Видно, что лидером стала вторая нода pg-srv02. Возвращаем в строй первую ноду, включаем питание.
Через минуту на второй ноде проверяем состояние кластера:
```console
yc-user@pg-srv02:~$ sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | streaming | 16 |   0/F0015C0 |   0 |  0/F0015C0 |   0 |
| pg-srv02 | 192.168.0.11 | Leader  | running   | 16 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming | 16 |   0/F0015C0 |   0 |  0/F0015C0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```

Первая нода успешно запустилась (сама, значит мы всё правильно настроили) и автоматически вернулась в кластер в роли реплики.

**Испытание failover проведено успешно.**


## 9. Воозможные ошибки

**9.1 Patroni запустился на первой ноде, но с ролью реплика.**

```console
+ Cluster: pg-cluster (7604445741021863162) ---+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State    | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | starting |    |     unknown |     |    unknown |     |
+----------+--------------+---------+----------+----+-------------+-----+------------+-----+
```
Смотрим журнал
```bash
sudo journalctl -u patroni -f
```
```console
yc-user@pg-srv01:~$ sudo journalctl -u patroni -f
Feb 08 11:46:45 pg-srv01 patroni[13174]: 2026-02-08 11:46:45,520 WARNING: Failed to determine PostgreSQL state from the connection, falling back to cached role
Feb 08 11:46:45 pg-srv01 patroni[13174]: 2026-02-08 11:46:45,564 INFO: Error communicating with PostgreSQL. Will try again later
Feb 08 11:46:55 pg-srv01 patroni[13386]: 127.0.0.1:5432:5432 - no response
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,148 INFO: Lock owner: None; I am pg-srv01
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,148 INFO: Still starting up as a standby.
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,148 INFO: establishing a new patroni heartbeat connection to postgres
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,939 INFO: establishing a new patroni heartbeat connection to postgres
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,940 WARNING: Retry got exception: connection problems
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,940 WARNING: Failed to determine PostgreSQL state from the connection, falling back to cached role
Feb 08 11:46:55 pg-srv01 patroni[13174]: 2026-02-08 11:46:55,985 INFO: Error communicating with PostgreSQL. Will try again later
Feb 08 11:47:05 pg-srv01 patroni[13393]: 127.0.0.1:5432:5432 - no response
```
В этом случае допущена ошибка в конфигурации /etc/postgresql/18/main/pg_hba.conf, не прописаны локальные подключения через сокет. Patroni не смог подключиться к PostgreSQL.
Проверить подключения в разных режимах можно командами:
Проверяем на первой ноде подключение от пользователей, локально и по tcp:
```bash
sudo -u postgres psql -c "SELECT 'Socket connection via peer auth: OK' as status;"
sudo -u postgres psql -U postgres -c "SELECT 'Socket connection via scram-sha-256: OK' as status;"
sudo -u postgres psql -h 127.0.0.1 -U postgres -c "SELECT 'TCP localhost connection: OK' as status;"
sudo -u postgres psql -h 192.168.0.10 -U postgres -c "SELECT 'TCP node IP connection: OK' as status;"
sudo -u postgres psql -c "SELECT 'Replicator user exists:' as check, usename FROM pg_user WHERE usename = 'replicator';"
sudo -u postgres psql -h 192.168.0.10 -U replicator -d postgres -c "SELECT 'Replicator TCP connection: OK' as status;"
```

**9.2 При смене лидера, лидер изменился, но нода, которая раньше была лидером, не перешла в состояние streaming, а находится в running**

Сделан switchower, анализируется состояние кластера. Прежний лидер pg-srv01, новый лидер pg-srv02. pg-srv01 не получает изменений с pg-srv02, не переходит в состояние "streaming".

```bash
sudo -u postgres /opt/patroni/venv/bin/patronictl -c /etc/patroni/patroni.yml list
```
```console
+ Cluster: pg-cluster (7604470388804299721) ----+----+-------------+-----+------------+-----+
| Member   | Host         | Role    | State     | TL | Receive LSN | Lag | Replay LSN | Lag |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
| pg-srv01 | 192.168.0.10 | Replica | running   |    |   0/5000000 |   0 |  0/5000560 |   0 |
| pg-srv02 | 192.168.0.11 | Leader  | running   |  3 |             |     |            |     |
| pg-srv03 | 192.168.0.12 | Replica | streaming |  3 |   0/50006A0 |   0 |  0/50006A0 |   0 |
+----------+--------------+---------+-----------+----+-------------+-----+------------+-----+
```
Проблема в конфигурационном файле patroni.yml на pg-srv02 и pg-srv03. В конфигах отсуствовала секция bootstrap, и не было прописано подключение в 192.168.0.10 в секции pg_hba.

**9.3 Patroni и Postgres читают конфиги из разных мест**

Debian/Ubuntu хранят конфиги PostgreSQL в двух местах:
/etc/postgresql/18/main/ — для systemd сервиса postgresql
/var/lib/postgresql/18/main/ — для Patroni и ручного запуска

Patroni создает pg_hba.conf в data_dir (/var/lib/postgresql/18/main/), но PostgreSQL может читать из /etc/postgresql/18/main/

Посмотреть, какой файл используется:
```bash
sudo -u postgres psql -c "SHOW hba_file; SHOW config_file;"
```

Решение:

**остановить patroni**
```bash
sudo systemctl stop patroni
```

**пророверить реальные пути:**
```bash
sudo ls -la /etc/postgresql/18/main/postgresql.conf
sudo ls -la /var/lib/postgresql/18/main/postgresql.conf
```

**сделать единую точку управления**
```bash
sudo mv /etc/postgresql/18/main/pg_hba.conf /etc/postgresql/18/main/pg_hba.conf.backup
sudo ln -sf /var/lib/postgresql/18/main/pg_hba.conf /etc/postgresql/18/main/pg_hba.conf
```

**сделать то же для postgresql.conf**
```bash
sudo mv /etc/postgresql/18/main/postgresql.conf /etc/postgresql/18/main/postgresql.conf.backup
sudo ln -sf /var/lib/postgresql/18/main/postgresql.conf /etc/postgresql/18/main/postgresql.conf
```

**запустить patroni**
```bash
sudo systemctl start patroni
```

**проверить откуда postgres читает конфиги и что в них**
```bash
sudo -u postgres psql -c "SHOW hba_file; SHOW config_file;"
```


**Что сделано:**

- созданы симлинки - /etc/postgresql/18/main/pg_hba.conf → /var/lib/postgresql/18/main/pg_hba.conf
- postgres теперь читает конфиги из /etc/, но через симлинки фактически использует файлы из data_dir
- patroni запустился успешно - реплика работает, WAL streaming идет
- postgres показывает правильные пути: /etc/postgresql/18/main/pg_hba.conf

**9.4 Проблема - patroni не мог подключится к postgres после того, как становился лидером**

PostgreSQL слушал только на внешнем IP (192.168.0.12), но не на 127.0.0.1.

Исправлены patroni.yml, добавлено прослушивание 127.0.0.1 к основному адресу, на всех нодах:

  listen: 127.0.0.1,192.168.0.10:5432

**9.5 Конфликт аутентификации**

В patroni.yml был режим scram-sha-256

В /etc/postgresql/18/main/pg_hba.conf был md5

На всех узлах кластера /etc/postgresql/18/main/pg_hba.conf приведен в общее состояние. Добавлены записи для пользователя replicator.
Внесены изменения в /etc/patroni/patroni.yml, указан режим аутентификации md5 в секции pg_hba:
```
  pg_hba:
    - host replication replicator 192.168.0.10/32 md5
    - host replication replicator 192.168.0.11/32 md5
    - host replication replicator 192.168.0.12/32 md5
    - host all all 192.168.0.0/24 md5
```

```bash
sudo cat /etc/postgresql/18/main/pg_hba.conf
```
```
local   all             all                                     trust
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
host    all             all             192.168.0.0/24          md5
host    replication     replicator      192.168.0.10/32         md5
host    replication     replicator      192.168.0.11/32         md5
host    replication     replicator      192.168.0.12/32         md5
```