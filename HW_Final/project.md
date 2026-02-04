### Подключаемся к Яндекс.Облако и выполняем конфигурацию окружения с помощью команды:
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

Создаём сеть
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

Создаём подсеть
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

Нарезаем виртуальные машинки (ВМ). Всего в кластере будет 5 ВМ:

- 3 ВМ под postgres с Patroni, etcd и PgBouncer
- 2 ВМ с HAProxy+KeepAlived

В качестве операционной системы на всех ВМ ставим Ubuntu 24.04 lts.

ВМ 1:
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
done (47s)
id: fhmnqrtj07lpu53ck22c
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-04T08:50:52Z"
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
  device_name: fhmlu992m6q75g3dajl1
  auto_delete: true
  disk_id: fhmlu992m6q75g3dajl1
network_interfaces:
  - index: "0"
    mac_address: d0:0d:17:d6:fb:30
    subnet_id: e9b57fk4u523knrmm4e7
    primary_v4_address:
      address: 192.168.0.10
      one_to_one_nat:
        address: 93.77.178.251
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
ВМ 2:
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
done (1m44s)
id: fhmg84v85vak7l3udisd
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-04T09:08:20Z"
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
  device_name: fhm199ekpg9audb23vf8
  auto_delete: true
  disk_id: fhm199ekpg9audb23vf8
network_interfaces:
  - index: "0"
    mac_address: d0:0d:10:41:3e:82
    subnet_id: e9b57fk4u523knrmm4e7
    primary_v4_address:
      address: 192.168.0.11
      one_to_one_nat:
        address: 93.77.183.76
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
ВМ 3:
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
done (42s)
id: fhmk55i9avqq89inp6nm
folder_id: b1go1kj72bcqdmjp8r7p
created_at: "2026-02-04T09:04:09Z"
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
  device_name: fhmorfstmgdl0otm8grg
  auto_delete: true
  disk_id: fhmorfstmgdl0otm8grg
network_interfaces:
  - index: "0"
    mac_address: d0:0d:14:29:64:95
    subnet_id: e9b57fk4u523knrmm4e7
    primary_v4_address:
      address: 192.168.0.12
      one_to_one_nat:
        address: 93.77.188.41
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

список готовых инстансов:
```bash
yc compute instance list
```
```console
+----------------------+----------+---------------+---------+---------------+--------------+
|          ID          |   NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP  |
+----------------------+----------+---------------+---------+---------------+--------------+
| fhmg84v85vak7l3udisd | pg-srv02 | ru-central1-a | RUNNING | 93.77.183.76  | 192.168.0.11 |
| fhmk55i9avqq89inp6nm | pg-srv03 | ru-central1-a | RUNNING | 93.77.188.41  | 192.168.0.12 |
| fhmnqrtj07lpu53ck22c | pg-srv01 | ru-central1-a | RUNNING | 93.77.178.251 | 192.168.0.10 |
+----------------------+----------+---------------+---------+---------------+--------------+
```

Подключаемся к машинкам:
```bash
ssh -i ~/.ssh/yc_key yc-user@93.77.178.251
ssh -i ~/.ssh/yc_key yc-user@93.77.183.76
ssh -i ~/.ssh/yc_key yc-user@93.77.188.41
```

Устанавливаем обновления и базовый набор инструментов на все машины:
```bash
sudo apt update &&
sudo apt upgrade -y &&
sudo apt install -y curl wget zip unzip tar nano htop neofetch &&
sudo apt autoremove -y
```

### Установка ETCD

Ставим и настраиваем etcd на первом сервере, pg-srv01:
```bash
sudo apt -y install etcd-server
sudo apt -y install etcd-client
```

Останавливаем etcd:
```bash
sudo systemctl stop etcd
sudo systemctl disable etcd
```

Удаляем дефолтный конфиг etcd
```bash
sudo rm -rf /var/lib/etcd/default
```

Создаём конфиг на первом хосте etcd, ВМ pg-srv01:
```bash
sudo vi /etc/default/etcd
```

```
ETCD_NAME="etcd-1"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.10:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.10:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.10:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.10:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_OTUS_Claster"
ETCD_INITIAL_CLUSTER="etcd-1=http://192.168.0.10:2380,etcd-2=http://192.168.0.11:2380,etcd-3=http://192.168.0.12:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

``bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

Кластер не запускается:
```console
Job for etcd.service failed because a timeout was exceeded.
See "systemctl status etcd.service" and "journalctl -xeu etcd.service" for details.
```
смотрим, почему:
```bash
sudo systemctl status etcd.service
```
```console
● etcd.service - etcd - highly-available key value store
     Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled; preset: enabled)
     Active: activating (start) since Wed 2026-02-04 09:50:59 UTC; 12s ago
       Docs: https://etcd.io/docs
             man:etcd
   Main PID: 10954 (etcd)
      Tasks: 7 (limit: 4653)
     Memory: 24.8M (peak: 25.1M)
        CPU: 296ms
     CGroup: /system.slice/etcd.service
             └─10954 /usr/bin/etcd

Feb 04 09:50:59 pg-srv01 etcd[10954]: started streaming with peer 951396625cd7422d (stream MsgApp v2 reader)
Feb 04 09:50:59 pg-srv01 etcd[10954]: added member ac6c070e1914f946 [http://192.168.0.10:2380] to cluster 3c403deb250ea5a5
Feb 04 09:51:04 pg-srv01 etcd[10954]: health check for peer 78c7ce11a1ae7376 could not connect: dial tcp 192.168.0.11:2380: connect: connection refused
Feb 04 09:51:04 pg-srv01 etcd[10954]: health check for peer 78c7ce11a1ae7376 could not connect: dial tcp 192.168.0.11:2380: connect: connection refused
Feb 04 09:51:04 pg-srv01 etcd[10954]: health check for peer 951396625cd7422d could not connect: dial tcp 192.168.0.12:2380: connect: connection refused
Feb 04 09:51:04 pg-srv01 etcd[10954]: health check for peer 951396625cd7422d could not connect: dial tcp 192.168.0.12:2380: connect: connection refused
Feb 04 09:51:09 pg-srv01 etcd[10954]: health check for peer 78c7ce11a1ae7376 could not connect: dial tcp 192.168.0.11:2380: connect: connection refused
Feb 04 09:51:09 pg-srv01 etcd[10954]: health check for peer 78c7ce11a1ae7376 could not connect: dial tcp 192.168.0.11:2380: connect: connection refused
Feb 04 09:51:09 pg-srv01 etcd[10954]: health check for peer 951396625cd7422d could not connect: dial tcp 192.168.0.12:2380: connect: connection refused
Feb 04 09:51:09 pg-srv01 etcd[10954]: health check for peer 951396625cd7422d could not connect: dial tcp 192.168.0.12:2380: connect: connection refused
```
Все потому что остальных хостов в кластере ещё нет. Логично

Сделаем два оставшихся узла кластера etcd. Повторить установку etsd на хостах pg-srv02, pg-srv03

Создаём конфиг на первом хосте etcd, ВМ pg-srv02:
```bash
sudo vi /etc/default/etcd
```
```
ETCD_NAME="etcd-2"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.11:2379,http://localhost:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.11:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.11:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.11:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd_OTUS_Claster"
ETCD_INITIAL_CLUSTER="etcd-1=http://192.168.0.10:2380,etcd-2=http://192.168.0.11:2380,etcd-3=http://192.168.0.12:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```



проверить, слушает ли etcd порт:
```bash
sudo ss -tlnp | grep 2380
```


Состав и состояние кластера
```bash
etcdctl endpoint status --cluster -w table
```
```console
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|         ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| http://192.168.0.11:2379 | 78c7ce11a1ae7376 |  3.4.30 |   20 kB |      true |      false |        94 |         11 |                 11 |        |
| http://192.168.0.12:2379 | 951396625cd7422d |  3.4.30 |   20 kB |     false |      false |        94 |         11 |                 11 |        |
| http://192.168.0.10:2379 | ac6c070e1914f946 |  3.4.30 |   20 kB |     false |      false |        94 |         11 |                 11 |        |
+--------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

Кластер etcd работает, лидер выбран, ошибок нет.

Устанавливаем postgres на всех хостах

sudo apt update && \
sudo apt upgrade -y && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt-get -y install postgresql && \
sudo apt install unzip






Удаляем машины, подсеть и сеть. Потом доделаем
yc compute instance delete pg-srv01
yc compute instance delete pg-srv02
yc compute instance delete pg-srv03
yc vpc subnet delete otus-pg-subnet
yc vpc network delete otus-pg-net