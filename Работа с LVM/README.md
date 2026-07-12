# Домашнее задание: Файловые системы и LVM

## 1\. Уменьшение корня (/) до 8G

*Выполняем перенос системы на временный том, чтобы уменьшить основной.*

```bash
kirills@otus:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 11.5G  0 lvm  /
sdb                         8:16   0   10G  0 disk
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
sr0                        11:0    1 1024M  0 rom

sudo -i
# Подготавливаем временный PV и VG на диске sdb
root@otus:~# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
root@otus:~# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
root@otus:~# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
root@otus:~# mkfs.ext4 /dev/vg_root/lv_root
mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 2621440 4k blocks and 655360 inodes
Filesystem UUID: b0529421-6b13-42b6-b832-e63e94d87319
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done


# Копируем систему
mount /dev/vg_root/lv_root /mnt
rsync -avxHAX --progress / /mnt/

# Настройка GRUB для загрузки с нового раздела
for i in /proc/ /sys/ /dev/ /run/ /boot/; do sudo mount --bind $i /mnt/$i; done
chroot /mnt/
grub-mkconfig -o /boot/grub/grub.cfg
update-initramfs -u
exit
reboot
# Перезагрузка: убедиться, что в BIOS/UEFI выбран загрузчик с sdb

kirills@otus:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part
  └─ubuntu--vg-ubuntu--lv 252:1    0 11.5G  0 lvm
sdb                         8:16   0   10G  0 disk
└─vg_root-lv_root         252:0    0   10G  0 lvm  /
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
sde                         8:64   0    1G  0 disk
sr0                        11:0    1 1024M  0 rom
kirills@otus:~$

```

*После перезагрузки удаляем старый корень и создаем его размером 8G:*

```
kirills@otus:~$ sudo -i
root@otus:~# lvremove /dev/ubuntu-vg/ubuntu-lv
Do you really want to remove and DISCARD active logical volume ubuntu-vg/ubuntu-lv? [y/n]: y
  Logical volume "ubuntu-lv" successfully removed.

root@otus:~# lvcreate -n ubuntu-lv -L 8G /dev/ubuntu-vg
WARNING: ext4 signature detected on /dev/ubuntu-vg/ubuntu-lv at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/ubuntu-vg/ubuntu-lv.
  Logical volume "ubuntu-lv" created.

root@otus:~# mkfs.ext4 /dev/ubuntu-vg/ubuntu-lv
mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 2097152 4k blocks and 524288 inodes
Filesystem UUID: d2e33be4-9f37-4708-88db-386599ef3b6d
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done

mount /dev/ubuntu-vg/ubuntu-lv /mnt
rsync -avxHAX --progress / /mnt/
# Снова настраиваем grub (как в блоке выше)
# Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести /var.

```

## 2\. Выделение тома под /var (Mirror)

*Используем диски sdd и sde.*

```bash
root@otus:/# pvcreate /dev/sdd /dev/sde
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.

root@otus:/# vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created

root@otus:/# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.

root@otus:/# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.47.2 (1-Jan-2025)
Creating filesystem with 243712 4k blocks and 60928 inodes
Filesystem UUID: 3a66d819-383c-438f-9563-6db054a42e6d
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done


# Перенос данных
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/
umount /mnt
mount /dev/vg_var/lv_var /var

# Добавляем в fstab (узнать UUID через blkid)
root@otus:/# blkid
/dev/mapper/ubuntu--vg-ubuntu--lv: UUID="d2e33be4-9f37-4708-88db-386599ef3b6d" BLOCK_SIZE="4096" TYPE="ext4"
/dev/sda3: UUID="sTJgT7-YYhn-c1wP-gerG-HOV6-aix9-D9pdwj" TYPE="LVM2_member" PARTUUID="36fbadcb-7423-4e67-bad5-99a0836f7461"
/dev/sda2: UUID="45a3ce8b-1bff-48dd-831a-ff213f5f992e" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="830b7d3a-363d-48ee-9ccd-d7dbbd6d18c2"
/dev/sdd: UUID="9V4w3S-3j9g-KQ0z-Vgu2-qB6N-6Veh-DnWAGf" TYPE="LVM2_member"
/dev/sdb: UUID="dif33o-1YBd-tOzz-wUAH-i1la-YooN-NwGgT5" TYPE="LVM2_member"
/dev/mapper/vg_var-lv_var: UUID="3a66d819-383c-438f-9563-6db054a42e6d" BLOCK_SIZE="4096" TYPE="ext4"
/dev/mapper/vg_root-lv_root: UUID="b0529421-6b13-42b6-b832-e63e94d87319" BLOCK_SIZE="4096" TYPE="ext4"
/dev/sde: UUID="rFoTI5-i8hy-pi7A-e7Sj-MAA7-yivw-6XLcRh" TYPE="LVM2_member"
/dev/sda1: PARTUUID="cbd81eeb-45ef-45bf-8001-e150da39655c"
/dev/mapper/vg_var-lv_var_rimage_1: UUID="3a66d819-383c-438f-9563-6db054a42e6d" BLOCK_SIZE="4096" TYPE="ext4"
/dev/mapper/vg_var-lv_var_rimage_0: UUID="3a66d819-383c-438f-9563-6db054a42e6d" BLOCK_SIZE="4096" TYPE="ext4"

echo "$(blkid | grep lv_var | awk '{print $2}') /var ext4 defaults 0 0" | sudo tee -a /etc/fstab
echo "UUID=3a66d819-383c-438f-9563-6db054a42e6d /var ext4 defaults 0 2" | tee -a /etc/fstab

root@otus:/# echo "UUID=3a66d819-383c-438f-9563-6db054a42e6d /var ext4 defaults 0 2" | tee -a /etc/fstab
UUID=3a66d819-383c-438f-9563-6db054a42e6d /var ext4 defaults 0 2
root@otus:/# cat /etc/fstab
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/ubuntu-vg/ubuntu-lv during curtin installation
/dev/disk/by-id/dm-uuid-LVM-EVADbvgowIfDnfNQ0Xf3grgn6TApu2BZsNMQkzp93ljl96Hd3UdOBo2plAJ9hQ8d / ext4 defaults 0 1
# /boot was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/45a3ce8b-1bff-48dd-831a-ff213f5f992e /boot ext4 defaults 0 1
/swap.img       none    swap    sw      0       0
UUID=3a66d819-383c-438f-9563-6db054a42e6d /var ext4 defaults 0 2

# перезагружаемся в новый (уменьшенный root) и удалять временную Volume Group:

kirills@otus:~$ sudo -i
[sudo: authenticate] Password:
root@otus:~# lvremove /dev/vg_root/lv_root
Do you really want to remove and DISCARD active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed.

root@otus:~# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed

root@otus:~# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
  
 root@otus:~# lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0    8G  0 lvm  /
sdb                         8:16   0   10G  0 disk
sdc                         8:32   0    2G  0 disk
sdd                         8:48   0    1G  0 disk
├─vg_var-lv_var_rmeta_0   252:2    0    4M  0 lvm
│ └─vg_var-lv_var         252:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0  252:3    0  952M  0 lvm
  └─vg_var-lv_var         252:6    0  952M  0 lvm  /var
sde                         8:64   0    1G  0 disk
├─vg_var-lv_var_rmeta_1   252:4    0    4M  0 lvm
│ └─vg_var-lv_var         252:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1  252:5    0  952M  0 lvm
  └─vg_var-lv_var         252:6    0  952M  0 lvm  /var
sr0                        11:0    1 1024M  0 rom


```

## 3\. Выделение тома под /home

*Используем оставшееся место в ubuntu-vg.*

```bash
root@otus:~# lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
  Logical volume "LogVol_Home" created.

root@otus:~# lvcreate -n LogVol_Home -L 2G /dev/ubuntu-vg
  Logical volume "LogVol_Home" created.
root@otus:~# mkfs.xfs /dev/ubuntu-vg/LogVol_Home
meta-data=/dev/ubuntu-vg/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=0   metadir=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
         =                       rgcount=0    rgsize=0 extents
         =                       zoned=0      start=0 reserved=0

mount /dev/ubuntu-vg/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
mount /dev/ubuntu-vg/LogVol_Home /home/
umount /mnt

# Добавляем в fstab
echo "$(blkid | grep LogVol_Home | awk '{print $2}') /home xfs defaults 0 0" | sudo tee -a /etc/fstab
```

## 4\. Работа со снапшотами

```bash
# Генерация файлов
touch /home/file{1..20}

root@otus:~# ls -la /home/
total 4
drwxr-xr-x  3 root    root     292 Jul 12 20:44 .
drwxr-xr-x 21 root    root    4096 Jul  9 15:05 ..
-rw-r--r--  1 root    root       0 Jul 12 20:44 file1
-rw-r--r--  1 root    root       0 Jul 12 20:44 file10
-rw-r--r--  1 root    root       0 Jul 12 20:44 file11
-rw-r--r--  1 root    root       0 Jul 12 20:44 file12
-rw-r--r--  1 root    root       0 Jul 12 20:44 file13
-rw-r--r--  1 root    root       0 Jul 12 20:44 file14
-rw-r--r--  1 root    root       0 Jul 12 20:44 file15
-rw-r--r--  1 root    root       0 Jul 12 20:44 file16
-rw-r--r--  1 root    root       0 Jul 12 20:44 file17
-rw-r--r--  1 root    root       0 Jul 12 20:44 file18
-rw-r--r--  1 root    root       0 Jul 12 20:44 file19
-rw-r--r--  1 root    root       0 Jul 12 20:44 file2
-rw-r--r--  1 root    root       0 Jul 12 20:44 file20
-rw-r--r--  1 root    root       0 Jul 12 20:44 file3
-rw-r--r--  1 root    root       0 Jul 12 20:44 file4
-rw-r--r--  1 root    root       0 Jul 12 20:44 file5
-rw-r--r--  1 root    root       0 Jul 12 20:44 file6
-rw-r--r--  1 root    root       0 Jul 12 20:44 file7
-rw-r--r--  1 root    root       0 Jul 12 20:44 file8
-rw-r--r--  1 root    root       0 Jul 12 20:44 file9
drwxr-x---  4 kirills kirills  123 Jul 12 13:33 kirills


# Создание снапшота
root@otus:~# lvcreate -L 500M -s -n home_snap /dev/ubuntu-vg/LogVol_Home
  Logical volume "home_snap" created.

root@otus:~# lvs
  LV          VG        Attr       LSize   Pool Origin      Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol_Home ubuntu-vg owi-aos---   2.00g
  home_snap   ubuntu-vg swi-a-s--- 500.00m      LogVol_Home 0.00
  ubuntu-lv   ubuntu-vg -wi-ao----   8.00g
  lv_var      vg_var    rwi-aor--- 952.00m                                         100.00


# Удаление файлов и восстановление
rm -f /home/file{11..20}

root@otus:~# ls -la /home/
total 4
drwxr-xr-x  3 root    root     152 Jul 12 20:47 .
drwxr-xr-x 21 root    root    4096 Jul  9 15:05 ..
-rw-r--r--  1 root    root       0 Jul 12 20:44 file1
-rw-r--r--  1 root    root       0 Jul 12 20:44 file10
-rw-r--r--  1 root    root       0 Jul 12 20:44 file2
-rw-r--r--  1 root    root       0 Jul 12 20:44 file3
-rw-r--r--  1 root    root       0 Jul 12 20:44 file4
-rw-r--r--  1 root    root       0 Jul 12 20:44 file5
-rw-r--r--  1 root    root       0 Jul 12 20:44 file6
-rw-r--r--  1 root    root       0 Jul 12 20:44 file7
-rw-r--r--  1 root    root       0 Jul 12 20:44 file8
-rw-r--r--  1 root    root       0 Jul 12 20:44 file9
drwxr-x---  4 kirills kirills  123 Jul 12 13:33 kirills

umount /home
root@otus:~# lvconvert --merge /dev/ubuntu-vg/home_snap
  Merging of volume ubuntu-vg/home_snap started.
  ubuntu-vg/LogVol_Home: Merged: 100.00%

mount /dev/ubuntu-vg/LogVol_Home /home
# Файлы вернулись
root@otus:~# ls -al /home
total 4
drwxr-xr-x  3 root    root     292 Jul 12 20:44 .
drwxr-xr-x 21 root    root    4096 Jul  9 15:05 ..
-rw-r--r--  1 root    root       0 Jul 12 20:44 file1
-rw-r--r--  1 root    root       0 Jul 12 20:44 file10
-rw-r--r--  1 root    root       0 Jul 12 20:44 file11
-rw-r--r--  1 root    root       0 Jul 12 20:44 file12
-rw-r--r--  1 root    root       0 Jul 12 20:44 file13
-rw-r--r--  1 root    root       0 Jul 12 20:44 file14
-rw-r--r--  1 root    root       0 Jul 12 20:44 file15
-rw-r--r--  1 root    root       0 Jul 12 20:44 file16
-rw-r--r--  1 root    root       0 Jul 12 20:44 file17
-rw-r--r--  1 root    root       0 Jul 12 20:44 file18
-rw-r--r--  1 root    root       0 Jul 12 20:44 file19
-rw-r--r--  1 root    root       0 Jul 12 20:44 file2
-rw-r--r--  1 root    root       0 Jul 12 20:44 file20
-rw-r--r--  1 root    root       0 Jul 12 20:44 file3
-rw-r--r--  1 root    root       0 Jul 12 20:44 file4
-rw-r--r--  1 root    root       0 Jul 12 20:44 file5
-rw-r--r--  1 root    root       0 Jul 12 20:44 file6
-rw-r--r--  1 root    root       0 Jul 12 20:44 file7
-rw-r--r--  1 root    root       0 Jul 12 20:44 file8
-rw-r--r--  1 root    root       0 Jul 12 20:44 file9
drwxr-x---  4 kirills kirills  123 Jul 12 13:33 kirills

```