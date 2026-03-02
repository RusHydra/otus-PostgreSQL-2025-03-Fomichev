## Домашнее задание 3. Установка и настройка PostgreSQL

### Cоздайте виртуальную машину c Ubuntu 20.04/22.04 LTS в ЯО/Virtual Box/докере
Машина создана, сделан проброс порта на 2222, подключаемся:
```bash
ssh -p 2222 student@127.0.0.1
```
```console
student@student-VirtualBox:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.3 LTS
Release:        24.04
Codename:       noble
```

### поставьте на нее PostgreSQL 15 через sudo apt
```console
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt-get -y install postgresql
```

### проверьте что кластер запущен через sudo -u postgres pg_lsclusters
```console
student@student-VirtualBox:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

### зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```console
student@student-VirtualBox:~$ sudo -u postgres psql -U postgres
psql (18.3 (Ubuntu 18.3-1.pgdg24.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
```

### остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
```console
student@student-VirtualBox:~$ sudo -u postgres pg_ctlcluster 18 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@18-main
student@student-VirtualBox:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

### создайте новый диск к ВМ размером 10GB
Создан

### добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
Готово

### проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux

Диск подмонтирован:
```console
student@student-VirtualBox:~$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       9.8G   24K  9.3G   1% /mnt/data
```
```console
student@student-VirtualBox:~$ sudo mkdir -p /mnt/data
student@student-VirtualBox:~$ sudo mount -o defaults /dev/sdb1 /mnt/data
student@student-VirtualBox:~$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       9.8G   24K  9.3G   1% /mnt/data
student@student-VirtualBox:~$ sudo blkid /dev/sdb1
/dev/sdb1: LABEL="mydata" UUID="5d58b783-675b-4a26-92c0-89b80bf281c1" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="primary" 

student@student-VirtualBox:~$ sudo nano /etc/fstab
```
Добавим строку в конец файла
UUID=5d58b783-675b-4a26-92c0-89b80bf281c1 /mnt/data ext4 defaults 0 2

### перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```console
sudo reboot
student@student-VirtualBox:~$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       9.8G   24K  9.3G   1% /mnt/data
```
### сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```console
sudo chown -R postgres:postgres /mnt/data/
```

### перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
```console
student@student-VirtualBox:/var/lib/postgres$ sudo mv /var/lib/postgresql/18/main /mnt/data/
student@student-VirtualBox:/var/lib/postgres$ sudo ls -ld /mnt/data/main
drwx------ 19 postgres postgres 4096 Mar  2 23:35 /mnt/data/main
```
### попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
```console
student@student-VirtualBox:/var/lib/postgres$ sudo -u postgres pg_ctlcluster 18 main start
Error: /var/lib/postgresql/18/main is not accessible or does not exist
```
### напишите получилось или нет и почему
Кластер не запустился, postgres всё ещё думает что data_directory находится по старому пути. Надо его образумить...

### задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
```console
sudo nano /etc/postgresql/18/main/postgresql.conf
```
Меняем 
data_directory = '/var/lib/postgresql/18/main'
на 
data_directory = '/mnt/data/main'

### попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
```console
student@student-VirtualBox:/var/lib/postgres$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory Log file
18  main    5432 online postgres /mnt/data/main /var/log/postgresql/postgresql-18-main.log
```
Кластер запустился

### зайдите через через psql и проверьте содержимое ранее созданной таблицы
```console
student@student-VirtualBox:/var/lib/postgres$ sudo -u postgres psql
psql (18.3 (Ubuntu 18.3-1.pgdg24.04+1))
Type "help" for help.
postgres=# SELECT * FROM test;
 c1
----
 1
(1 row)
```

Даные доступны, получилось.