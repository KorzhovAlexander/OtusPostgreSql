# Домашнее задание

## Установка и настройка PostgreSQL

### Цель:

- создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
- переносить содержимое базы данных PostgreSQL на дополнительный диск
- переносить содержимое БД PostgreSQL между виртуальными машинами

### Описание/Пошаговая инструкция выполнения домашнего задания:

- создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
- поставьте на нее PostgreSQL 15 через sudo apt
- проверьте что кластер запущен через sudo -u postgres pg_lsclusters
- зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
  postgres=# create table test(c1 text);
  postgres=# insert into test values('1');
  \q
- остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
- создайте новый диск к ВМ размером 10GB
- добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт
  attach existing disk
- проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на
  актуальное,
  в вашем случае это скорее всего будет
  /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
- перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
- сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
- перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
- попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
- напишите получилось или нет и почему
- задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и
  поменяйте его
- напишите что и почему поменяли
- попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
- напишите получилось или нет и почему
- зайдите через через psql и проверьте содержимое ранее созданной таблицы
- задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы
  с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко
  второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это
  сделали и что в итоге получилось.

Решение:

1) Создаю VM c Ubuntu 22.04 LTS.
2) Устанавливаю postgres

```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```

3) Проверяю что кластер запущен

```
korzhov@korzhov-postgres:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

4) Захожу в кластер и создаю таблицу, добавляю запись, делаю выборку по записям таблицы для проверки

```
korzhov@korzhov-postgres:~$ sudo -u postgres psql
could not change directory to "/home/korzhov": Permission denied
psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# select * from test;
 c1
----
 1
(1 row)
```

5) Остановливаю постгрес

```
korzhov@korzhov-postgres:~$ sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
korzhov@korzhov-postgres:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

6) Создал новый диск

```
Общая информация
Идентификатор 		- XXX
Имя			- korzhov-new-disk
Статус			- Ready
Размер			- 10 ГБ
Размер блока		- 4 КБ
Дата создания		- 27.01.2024, в 14:01
Тип			- SSD
Зона доступности	- ru-central1-a
```

7) Добавляю свеже-созданный диск к виртуальной машине

```
Общая информация
Виртуальные машины	- korzhov-postgres

korzhov@korzhov-postgres:/srv$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  61.9M  1 loop /snap/core20/1405
loop1    7:1    0  63.3M  1 loop /snap/core20/1852
loop2    7:2    0  79.9M  1 loop /snap/lxd/22923
loop3    7:3    0 111.9M  1 loop /snap/lxd/24322
loop4    7:4    0  44.7M  1 loop /snap/snapd/15534
vda    252:0    0    15G  0 disk
├─vda1 252:1    0     1M  0 part
└─vda2 252:2    0    15G  0 part /
vdb    252:16   0    10G  0 disk
```

8) Проинициализирую диск согласно инструкции и демонстрирую файловую систему

```
korzhov@korzhov-postgres:/srv$ sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME   FSTYPE   LABEL         UUID                                 MOUNTPOINT
loop0  squashfs                                                    /snap/core20/1405
loop1  squashfs                                                    /snap/core20/1852
loop2  squashfs                                                    /snap/lxd/22923
loop3  squashfs                                                    /snap/lxd/24322
loop4  squashfs                                                    /snap/snapd/15534
vda
├─vda1
└─vda2 ext4                   14aeea52-6d42-49e6-85d5-1071d3c9b2ab /
vdb
└─vdb1 ext4     datapartition a2a5c8a2-1c61-4632-b773-2acdca1fab12 /mnt/data

korzhov@korzhov-postgres:/srv$ df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        15G  5.3G  8.8G  37% /
/dev/vdb1       9.9G   28K  9.2G   2% /mnt/data
```

9) Перезапускаю инстанс и проверяю что диск примонтирован

```
korzhov@korzhov-postgres:~$ sudo lsblk -o NAME,FSTYPE,LABEL,UUID,MOUNTPOINT
NAME   FSTYPE   LABEL         UUID                                 MOUNTPOINT
loop0  squashfs                                                    /snap/core20/1852
loop1  squashfs                                                    /snap/core20/1405
loop2  squashfs                                                    /snap/lxd/22923
loop3  squashfs                                                    /snap/lxd/24322
loop4  squashfs                                                    /snap/snapd/15534
vda
├─vda1
└─vda2 ext4                   14aeea52-6d42-49e6-85d5-1071d3c9b2ab /
vdb
└─vdb1 ext4     datapartition a2a5c8a2-1c61-4632-b773-2acdca1fab12 /mnt/data
```

10) Сделал пользователя postgres владельцем /mnt/data

```
korzhov@korzhov-postgres:/mnt$ ls -l
total 4
drwxr-xr-x 3 postgres postgres 4096 Jan 27 14:10 data
```

11) Перенёс содержимое /var/lib/postgres/15 в /mnt/data

```
korzhov@korzhov-postgres:/mnt$ sudo mv /var/lib/postgresql/15 /mnt/data
korzhov@korzhov-postgres:/mnt$ cd data
korzhov@korzhov-postgres:/mnt/data$ ls -l
total 20
drwxr-xr-x 3 postgres postgres  4096 Jan 27 14:16 15
drwx------ 2 postgres postgres 16384 Jan 27 14:56 lost+found
```

12) Попытался запустить кластер - при запуске кластера получаю ошибку:

```
korzhov@korzhov-postgres:/mnt/data$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```

13) Ошибка связана с тем, что содержимое postgres было перенесено в другую папку. Папка пуста, а в настройках кластера указан старый путь

```
korzhov@korzhov-postgres:/mnt/data$ cd /var/lib/postgresql
korzhov@korzhov-postgres:/var/lib/postgresql$ ls -l
total 0
```

```
korzhov@korzhov-postgres:/var/lib/postgresql$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
15  main    5432 down   <unknown> /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

14) Найти файл /etc/postgresql/15/main/postgres.conf и поменять его
15) Поменять параметр data_directory на '/mnt/data'
16) Запуск кластера снова неудался
17) Ошибка

```
pg_ctl: directory "/mnt/data" is not a database cluster directory
```

Написал не полный путь, забыл добавить 15/main, путь должен выглядеть так: '/mnt/data/15/main'. После исправления
кластер запустился:

```
korzhov@korzhov-postgres:/etc/postgresql/15/main$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```

18) Подключился к БД, содержимое таблицы test осталось на месте

```
korzhov@korzhov-postgres:/etc/postgresql/15/main$ sudo -u postgres psql
psql (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
Type "help" for help.

postgres=# select * from test;
 c1
----
 1
(1 row)
```