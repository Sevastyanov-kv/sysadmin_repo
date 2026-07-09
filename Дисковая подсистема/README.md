Домашнее задание: работа с mdadm
Задание

Добавить в виртуальную машину несколько дисков

Собрать RAID-0/1/5/10 на выбор

Сломать и починить RAID

Создать GPT таблицу, пять разделов и смонтировать их в системе.

1. Проверка дисков и сборка RAID

Командой lsblk проверяем созданные HDD в виртуальной машине:

kirills@otus:~$ lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sda 8:0 0 25G 0 disk
├─sda1 8:1 0 1M 0 part
├─sda2 8:2 0 2G 0 part /boot
└─sda3 8:3 0 23G 0 part
  └─ubuntu--vg-ubuntu--lv 252:0 0 11.5G 0 lvm /
sdb 8:16 0 1G 0 disk
sdc 8:32 0 1G 0 disk
sdd 8:48 0 1G 0 disk
sde 8:64 0 1G 0 disk
sr0 11:0 1 1024M 0 rom

Удаляем метаданные RAID с указанных дисков:

sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e}

Примечание: В моем случае данную команду можно не выполнять, так как диски не использовались в RAID.

Создаем RAID-10 (-l 10), состоящий из 4 устройств (-n 4):

kirills@otus:~$ sudo mdadm --create --verbose /dev/md0 -l 10 -n 4 /dev/sd{b,c,d,e}
To optimize recovery speed, it is recommended to enable write-intent bitmap, do you want to enable it now? [y/N]? y
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: size set to 1046528K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
kirills@otus:~$

Проверим статус массивов следующими командами:

cat /proc/mdstat
sudo mdadm -D /dev/md0

Вывод команды cat /proc/mdstat:

kirills@otus:~$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sde[3] sdd[2] sdc[1] sdb[0]
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      bitmap: 0/1 pages [0KB], 65536KB chunk
unused devices: <none>

Вывод команды sudo mdadm -D /dev/md0:

kirills@otus:~$ sudo mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Jul 9 14:26:55 2026
        Raid Level : raid10
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent
     Intent Bitmap : Internal
       Update Time : Thu Jul 9 14:27:06 2026
             State : clean
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0
            Layout : near=2
        Chunk Size : 512K
Consistency Policy : bitmap
              Name : otus:0 (local to host otus)
              UUID : 491e928a:48cf2185:0d93ba62:903a20a4
            Events : 18

    Number Major Minor RaidDevice State
       0 8 16 0 active sync set-A /dev/sdb
       1 8 32 1 active sync set-B /dev/sdc
       2 8 48 2 active sync set-A /dev/sdd
       3 8 64 3 active sync set-B /dev/sde
2. Сломать и починить RAID

Ломаем: пометим диск /dev/sdb как неисправный:

sudo mdadm /dev/md0 --fail /dev/sdb

В выводе команды ниже виден помеченный диск sdb[0](F) — Faulty (неисправный):

kirills@otus:~$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sde[3] sdd[2] sdc[1] sdb[0](F)
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [_UUU]
      bitmap: 0/1 pages [0KB], 65536KB chunk
unused devices: <none>

Заметим, что несмотря на вышедший из строя диск /dev/sdb, RAID продолжает работать и данные продолжают читаться с диска. Чтобы восстановить избыточность массива, нам нужно удалить сломанный диск и заменить его на новый.

Удаляем сломанный диск из массива:

kirills@otus:~$ sudo mdadm /dev/md0 --remove /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md0

Проверяем статус:

kirills@otus:~$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sde[3] sdd[2] sdc[1]
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/3] [_UUU]
      bitmap: 0/1 pages [0KB], 65536KB chunk

Представим, что мы вставили новый диск в сервер и теперь нам нужно добавить его в RAID. Делается это так:

kirills@otus:~$ sudo mdadm /dev/md0 --add /dev/sdb
mdadm: re-added /dev/sdb

Снова проверяем состояние:

kirills@otus:~$ cat /proc/mdstat
Personalities : [raid10]
md0 : active raid10 sdb[0] sde[3] sdd[2] sdc[1]
      2093056 blocks super 1.2 512K chunks 2 near-copies [4/4] [UUUU]
      bitmap: 0/1 pages [0KB], 65536KB chunk
unused devices: <none>

Диск должен пройти стадию rebuilding. Например, если это был RAID-1 (зеркало), то данные должны скопироваться на новый диск. Процесс rebuild-а можно увидеть в выводе команд cat /proc/mdstat и mdadm -D /dev/md0.

На маленьких объемах занятого пространства можно и пропустить момент перестроения RAID-а — так быстро он проходит.

3. GPT, разделы и монтирование

Создаем таблицу разделов GPT на RAID-массиве:

sudo parted -s /dev/md0 mklabel gpt

Проверить создание GPT можно командой: sudo parted /dev/md0 print

kirills@otus:~$ sudo parted /dev/md0 print
Model: Linux Software RAID Array (md)
Disk /dev/md0: 2143MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number Start End Size File system Name Flags

Создаем 5 разделов по 20% пространства каждый:

sudo parted -s /dev/md0 mkpart primary ext4 0% 20%
sudo parted -s /dev/md0 mkpart primary ext4 20% 40%
sudo parted -s /dev/md0 mkpart primary ext4 40% 60%
sudo parted -s /dev/md0 mkpart primary ext4 60% 80%
sudo parted -s /dev/md0 mkpart primary ext4 80% 100%

Создаем файловую систему на каждом разделе и монтируем их:

# Создание ФС
for i in {1..5}; do sudo mkfs.ext4 /dev/md0p$i; done

# Создание точек монтирования и монтирование
sudo mkdir -p /raid/part{1..5}
for i in {1..5}; do sudo mount /dev/md0p$i /raid/part$i; done
Проверка результата

Проверяем конфигурацию блочных устройств через lsblk:

kirills@otus:~$ lsblk
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
sda 8:0 0 25G 0 disk
├─sda1 8:1 0 1M 0 part
├─sda2 8:2 0 2G 0 part /boot
└─sda3 8:3 0 23G 0 part
  └─ubuntu--vg-ubuntu--lv 252:0 0 11.5G 0 lvm /
sdb 8:16 0 1G 0 disk
└─md0 9:0 0 2G 0 raid10
  ├─md0p1 259:0 0 408M 0 part /raid/part1
  ├─md0p2 259:1 0 409M 0 part /raid/part2
  ├─md0p3 259:2 0 408M 0 part /raid/part3
  ├─md0p4 259:3 0 409M 0 part /raid/part4
  └─md0p5 259:4 0 408M 0 part /raid/part5
sdc 8:32 0 1G 0 disk
└─md0 9:0 0 2G 0 raid10
  ├─md0p1 259:0 0 408M 0 part /raid/part1
  ├─md0p2 259:1 0 409M 0 part /raid/part2
  ├─md0p3 259:2 0 408M 0 part /raid/part3
  ├─md0p4 259:3 0 409M 0 part /raid/part4
  └─md0p5 259:4 0 408M 0 part /raid/part5
sdd 8:48 0 1G 0 disk
└─md0 9:0 0 2G 0 raid10
  ├─md0p1 259:0 0 408M 0 part /raid/part1
  ├─md0p2 259:1 0 409M 0 part /raid/part2
  ├─md0p3 259:2 0 408M 0 part /raid/part3
  ├─md0p4 259:3 0 409M 0 part /raid/part4
  └─md0p5 259:4 0 408M 0 part /raid/part5
sde 8:64 0 1G 0 disk
└─md0 9:0 0 2G 0 raid10
  ├─md0p1 259:0 0 408M 0 part /raid/part1
  ├─md0p2 259:1 0 409M 0 part /raid/part2
  ├─md0p3 259:2 0 408M 0 part /raid/part3
  ├─md0p4 259:3 0 409M 0 part /raid/part4
  └─md0p5 259:4 0 408M 0 part /raid/part5
sr0 11:0 1 1024M 0 rom

Проверяем свободное место и точки монтирования:

kirills@otus:~$ df -h | grep /raid
/dev/md0p1 366M 152K 338M 1% /raid/part1
/dev/md0p2 367M 152K 339M 1% /raid/part2
/dev/md0p3 366M 152K 338M 1% /raid/part3
/dev/md0p4 367M 152K 339M 1% /raid/part4
/dev/md0p5 366M 152K 338M 1% /raid/part5
