# Домашнее задание: настройка NFS

## Цель

Развернуть сервис NFS и подключить к нему клиента, обеспечив автоматическое монтирование при старте системы (NFSv3), с возможностью записи в выделенную поддиректорию.

## Топология стенда

| Роль | Hostname | IP-адрес |
| --- | --- | --- |
| NFS-сервер | otus | 192.168.1.57 |
| NFS-клиент | otus1 | 192.168.1.38 |

* * *

## 1\. Настройка сервера NFS (192.168.1.57)

Все команды выполняются от пользователя с правами root (sudo).

### 1.1 Установка пакета

```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

### 1.2 Проверка слушающих портов

```bash
ss -tnplu
```

Должны присутствовать порты: `2049/tcp`, `2049/udp`, `111/tcp`, `111/udp`.

### 1.3 Подготовка экспортируемой директории

```bash
sudo mkdir -p /srv/share/upload
sudo chown -R nobody:nogroup /srv/share
sudo chmod 0777 /srv/share/upload
```

### 1.4 Настройка экспорта

```bash
sudo -i
cat << EOF > /etc/exports
/srv/share 192.168.1.38/32(rw,sync,root_squash)
EOF
```

### 1.5 Применение экспорта

```bash
root@otus:~# exportfs -r
root@otus:~# exportfs -s
/srv/share  192.168.1.35/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

```

* * *

## 2\. Настройка клиента NFS (192.168.1.38)

### 2.1 Установка пакета

```bash
sudo apt update
sudo apt install -y nfs-common
```

### 2.2 Настройка автомонтирования через fstab

```bash
kirills@otus1:~$ echo "192.168.1.57:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" | sudo tee -a /etc/fstab
192.168.1.57:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0

```

### 2.3 Применение конфигурации

```bash
kirills@otus1:~$ sudo systemctl daemon-reload
kirills@otus1:~$ sudo systemctl restart remote-fs.target
kirills@otus1:~$ sudo systemctl status remote-fs.target
● remote-fs.target - Remote File Systems
     Loaded: loaded (/usr/lib/systemd/system/remote-fs.target; enabled; preset: enabled)
     Active: active since Sat 2026-07-25 06:19:40 UTC; 9s ago
 Invocation: c66186a6ba834452ac430d0ac9492830
       Docs: man:systemd.special(7)

Jul 25 06:19:40 otus1 systemd[1]: Stopped target remote-fs.target - Remote File Systems.
Jul 25 06:19:40 otus1 systemd[1]: Stopping remote-fs.target - Remote File Systems...
Jul 25 06:19:40 otus1 systemd[1]: Reached target remote-fs.target - Remote File Systems.
kirills@otus1:~$

```

Монтирование произойдёт автоматически при первом обращении к `/mnt` благодаря автогенерируемым unit-файлам в `/run/systemd/generator/`.

### 2.4 Проверка монтирования

```bash
cd /mnt
mount | grep mnt
```

Ожидаемый вывод (важно наличие `vers=3`, подтверждающего использование NFSv3):

```bash
systemd-1 on /mnt type autofs (rw,relatime,fd=84,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=20000)
192.168.1.57:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=262144,wsize=262144,namlen=255,hard,fatal_neterrors=none,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.57,mountvers=3,mountport=56652,mountproto=udp,local_lock=none,addr=192.168.1.57,x-systemd.automount)
kirills@otus1:/mnt$

```

* * *

## 3\. Проверка работоспособности

### 3.1 Базовая проверка чтения/записи

На сервере:

```bash
cd /srv/share/upload
touch check_file
```

На клиенте:

```bash
kirills@otus1:/mnt$ cd /mnt/upload
kirills@otus1:/mnt/upload$ ls -l
total 0
-rw-r--r-- 1 root root 0 Jul 25 06:31 check_file                   # видим check_file
kirills@otus1:/mnt/upload$ touch client_file                       # проверяем запись с клиента
kirills@otus1:/mnt/upload$ ls -l                                   # видим client_file
total 0
-rw-r--r-- 1 root    root    0 Jul 25 06:31 check_file
-rw-rw-r-- 1 kirills kirills 0 Jul 25 06:31 client_file
kirills@otus1:/mnt/upload$

```

### 3.2 Проверка после перезагрузки клиента

```bash
Last login: Sat Jul 25 05:58:38 2026 from 192.168.1.37
kirills@otus1:~$ cd /mnt/upload
kirills@otus1:/mnt/upload$ ls -l
total 0
-rw-r--r-- 1 root    root    0 Jul 25 06:31 check_file
-rw-rw-r-- 1 kirills kirills 0 Jul 25 06:31 client_file
kirills@otus1:/mnt/upload$

```

### 3.3 Проверка после перезагрузки сервера

На сервере (в отдельном окне терминала):

```bash
Last login: Sat Jul 25 05:58:44 2026 from 192.168.1.37
kirills@otus:~$ ls -l /srv/share/upload
total 0
-rw-r--r-- 1 root    root    0 Jul 25 06:31 check_file
-rw-rw-r-- 1 kirills kirills 0 Jul 25 06:31 client_file
kirills@otus:~$ exportfs -s
exportfs: could not open /var/lib/nfs/.etab.lock for locking: errno 13 (Permission denied)
kirills@otus:~$ sudo exportfs -s
[sudo: authenticate] Password:
/srv/share  192.168.1.38/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
kirills@otus:~$ showmount -a 192.168.1.57
All mount points on 192.168.1.57:
192.168.1.38:/srv/share
kirills@otus:~$

```

### 3.4 Финальная проверка на клиенте

```bash
kirills@otus1:/mnt/upload$ showmount -a 192.168.1.57
All mount points on 192.168.1.57:
192.168.1.38:/srv/share
kirills@otus1:/mnt/upload$ mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=80,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=6928)
192.168.1.57:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=262144,wsize=262144,namlen=255,hard,fatal_neterrors=none,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.1.57,mountvers=3,mountport=56652,mountproto=udp,local_lock=none,addr=192.168.1.57,x-systemd.automount)
kirills@otus1:/mnt/upload$ cd /mnt/upload
kirills@otus1:/mnt/upload$ ls -l
total 0
-rw-r--r-- 1 root    root    0 Jul 25 06:31 check_file
-rw-rw-r-- 1 kirills kirills 0 Jul 25 06:31 client_file
kirills@otus1:/mnt/upload$ touch final_check
kirills@otus1:/mnt/upload$ ls -l
total 0
-rw-r--r-- 1 root    root    0 Jul 25 06:31 check_file
-rw-rw-r-- 1 kirills kirills 0 Jul 25 06:31 client_file
-rw-rw-r-- 1 kirills kirills 0 Jul 25 06:39 final_check
kirills@otus1:/mnt/upload$

```

Если все шаги прошли успешно — стенд работоспособен.

* * *

## 4\. Скрипты автоматизации

### 4.1 `nfss_script.sh` (выполняется на сервере, 192.168.1.57)

```bash
#!/usr/bin/env bash
set -euo pipefail

CLIENT_IP="192.168.1.38"
EXPORT_DIR="/srv/share"
UPLOAD_DIR="${EXPORT_DIR}/upload"

apt update
apt install -y nfs-kernel-server

mkdir -p "$UPLOAD_DIR"
chown -R nobody:nogroup "$EXPORT_DIR"
chmod 0777 "$UPLOAD_DIR"

cat << EOF > /etc/exports
${EXPORT_DIR} ${CLIENT_IP}/32(rw,sync,root_squash)
EOF

exportfs -r
exportfs -s
```

### 4.2 `nfsc_script.sh` (выполняется на клиенте, 192.168.1.38)

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVER_IP="192.168.1.57"
FSTAB_LINE="${SERVER_IP}:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0"

apt update
apt install -y nfs-common

if ! grep -qF "$FSTAB_LINE" /etc/fstab; then
    echo "$FSTAB_LINE" >> /etc/fstab
fi

systemctl daemon-reload
systemctl restart remote-fs.target

# триггерим automount
ls /mnt >/dev/null
```

&nbsp;