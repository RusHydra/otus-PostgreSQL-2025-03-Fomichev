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
yc vpc network create --name otus-pg-net --description "otus-pg-net"

Создаём подсетьсеть
yc vpc subnet create --name otus-pg-subnet --range 192.168.0.0/24 --network-name otus-pg-net --description "otus-pg-subnet"

Нарезаем машинки. Всего будет 5 серверов:
3 машины под postgres с Patroni, etcd и PgBouncer
2 машины с HAProxy+KeepAlived

Нарезаем машинки, ставим Ubuntu 24.04 lts:

yc compute instance create `
    --name pg-srv01 `
    --hostname pg-srv01 `
    --cores 2 `
    --memory 4 `
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts `
    --network-interface subnet-name=otus-pg-subnet,nat-ip-version=ipv4,ipv4-address=192.168.0.10 `
    --ssh-key yc_key.pub


yc compute instance create `
    --name pg-srv02 `
    --hostname pg-srv02 `
    --cores 2 `
    --memory 4 `
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts `
    --network-interface subnet-name=otus-pg-subnet,nat-ip-version=ipv4,ipv4-address=192.168.0.11 `
    --ssh-key yc_key.pub
	
yc compute instance create `
    --name pg-srv03 `
    --hostname pg-srv03 `
    --cores 2 `
    --memory 4 `
    --create-boot-disk size=20G,type=network-ssd,image-folder-id=standard-images,image-family=ubuntu-2404-lts `
    --network-interface subnet-name=otus-pg-subnet,nat-ip-version=ipv4,ipv4-address=192.168.0.12 `
    --ssh-key yc_key.pub


список готовых инстансов

yc compute instance list

PS C:\Users\kfomi> yc compute instance list
+----------------------+----------+---------------+---------+----------------+--------------+
|          ID          |   NAME   |    ZONE ID    | STATUS  |  EXTERNAL IP   | INTERNAL IP  |
+----------------------+----------+---------------+---------+----------------+--------------+
| fhm42a8rapt2q7qikunh | pg-srv01 | ru-central1-a | RUNNING | 51.250.72.69   | 192.168.0.10 |
| fhmlbatriai0b44lpjng | pg-srv03 | ru-central1-a | RUNNING | 158.160.45.13  | 192.168.0.12 |
| fhmsr6a5hlcrpk5d42ah | pg-srv02 | ru-central1-a | RUNNING | 178.154.253.96 | 192.168.0.11 |
+----------------------+----------+---------------+---------+----------------+--------------+

Подключаемся к машинкам:

ssh -i ~/.ssh/yc_key yc-user@51.250.72.69

Ставим и настраиваем etcd на первом сервере, pg-srv01:
sudo apt -y install etcd-server
sudo apt -y install etcd-client

Останавливаем etcd
sudo systemctl stop etcd
sudo systemctl disable etcd

Удаляем дефолтный конфиг etcd
sudo rm -rf /var/lib/etcd/default

Создаём конфиг:
sudo vi /etc/default/etcd

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


sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd

проверить, слушает ли etcd порт
sudo ss -tlnp | grep 2380

Состав и состояние кластера
etcdctl member list
etcdctl cluster-health


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