# Домашнее задание: Инициализация системы. Systemd

## Задание

1.  Написать service, который раз в 30 секунд мониторит лог на предмет ключевого слова (файл лога и ключевое слово задаются в `/etc/default`).
2.  Установить `spawn-fcgi` и создать unit-файл `spawn-fcgi.service` на основе init-скрипта.
3.  Доработать unit-файл Nginx (`nginx.service`) для запуска нескольких инстансов сервера с разными конфигами одновременно.

* * *

## 1\. Сервис мониторинга лога (watchlog)

### 1.1. Конфигурация в `/etc/default`

```bash
root@otus:~# cat > /etc/default/watchlog << 'EOF'
# Configuration file for my watchlog service
# Place it to /etc/default

# File and word in that file that we will be monitoring
WORD="ALERT"
LOG=/var/log/watchlog.log
EOF
root@otus:~# ls -la /etc/default/watchlog
-rw-r--r-- 1 root root 168 Aug  2 08:58 /etc/default/watchlog
root@otus:~#

```

> **Примечание: д**ля начала создаём файл с конфигурацией для сервиса в директории /etc/default - из неё сервис будет брать необходимые переменные.

### 1.2. Тестовый лог-файл

```bash
root@otus:~# cat > /var/log/watchlog.log << 'EOF'
2024-06-05 12:00:01 system started normally
2024-06-05 12:00:15 user login successful
2024-06-05 12:00:42 ALERT disk usage exceeded 90%
2024-06-05 12:01:03 background job completed
EOF

```

### 1.3. Скрипт мониторинга

```bash
cat > /opt/watchlog.sh << 'EOF'
#!/bin/bash

WORD=$1
LOG=$2
DATE=$(date)

if grep "$WORD" "$LOG" &> /dev/null
then
    logger "$DATE: I found word, Master!"
else
    exit 0
fi
EOF

chmod +x /opt/watchlog.sh
```

### 1.4. Unit-файл сервиса

```bash
cat > /etc/systemd/system/watchlog.service << 'EOF'
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
EOF
```

### 1.5. Unit-файл таймера

```bash
cat > /etc/systemd/system/watchlog.timer << 'EOF'
[Unit]
Description=Run watchlog script every 30 seconds

[Timer]
# Run every 30 seconds
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
EOF
```

### 1.6. Запуск и проверка

```bash
systemctl daemon-reload
systemctl start watchlog.timer
systemctl enable watchlog.timer
systemctl start watchlog.service
systemctl enable watchlog.service
root@otus:~# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 30 seconds
     Loaded: loaded (/etc/systemd/system/watchlog.timer; enabled; preset: enabled)
     Active: active (elapsed) since Sun 2026-08-02 09:10:25 UTC; 26s ago
 Invocation: f3f5b16b1a5349d5adf0bd3603bbdfe3
    Trigger: n/a
   Triggers: ● watchlog.service

Aug 02 09:10:25 otus systemd[1]: Started watchlog.timer - Run watchlog script every 30 seconds.
root@otus:~#


root@otus:~# systemctl status watchlog.service
○ watchlog.service - My watchlog service
     Loaded: loaded (/etc/systemd/system/watchlog.service; static)
     Active: inactive (dead) since Sun 2026-08-02 09:14:19 UTC; 21s ago
 Invocation: cc343c6057fe4afeb4fc348207756814
TriggeredBy: ● watchlog.timer
    Process: 2554 ExecStart=/opt/watchlog.sh $WORD $LOG (code=exited, status=0/SUCCESS)
   Main PID: 2554 (code=exited, status=0/SUCCESS)
   Mem peak: 2.3M
        CPU: 14ms

Aug 02 09:14:19 otus systemd[1]: Starting watchlog.service - My watchlog service...
Aug 02 09:14:19 otus root[2557]: Sun Aug  2 09:14:19 AM UTC 2026: I found word, Master!
Aug 02 09:14:19 otus systemd[1]: watchlog.service: Deactivated successfully.
Aug 02 09:14:19 otus systemd[1]: Finished watchlog.service - My watchlog service.

```

Проверка списка таймеров:

```bash
systemctl list-timers | grep watchlog
```

```
Sun 2026-08-02 09:20:34 UTC   15s Sun 2026-08-02 09:20:04 UTC   14s ago watchlog.timer   watchlog.service
```

Проверка журнала:

```bash
tail -n 1000 /var/log/syslog | grep "found word"
```

```
root@otus:~# tail -n 1000 /var/log/syslog | grep "found word"
2026-08-02T09:13:48.369671+00:00 otus root: Sun Aug  2 09:13:48 AM UTC 2026: I found word, Master!
2026-08-02T09:14:19.792194+00:00 otus root: Sun Aug  2 09:14:19 AM UTC 2026: I found word, Master!
2026-08-02T09:15:02.097981+00:00 otus root: Sun Aug  2 09:15:02 AM UTC 2026: I found word, Master!
2026-08-02T09:15:54.255335+00:00 otus root: Sun Aug  2 09:15:54 AM UTC 2026: I found word, Master!
2026-08-02T09:16:44.410353+00:00 otus root: Sun Aug  2 09:16:44 AM UTC 2026: I found word, Master!
2026-08-02T09:17:19.793426+00:00 otus root: Sun Aug  2 09:17:19 AM UTC 2026: I found word, Master!
2026-08-02T09:18:19.793672+00:00 otus root: Sun Aug  2 09:18:19 AM UTC 2026: I found word, Master!
2026-08-02T09:18:59.561731+00:00 otus root: Sun Aug  2 09:18:59 AM UTC 2026: I found word, Master!
2026-08-02T09:20:04.621495+00:00 otus root: Sun Aug  2 09:20:04 AM UTC 2026: I found word, Master!
2026-08-02T09:20:54.778403+00:00 otus root: Sun Aug  2 09:20:54 AM UTC 2026: I found word, Master!

```

Результат подтверждает корректную работу пары `.service` + `.timer`.

**Проверка статуса unit'ов:**

```bash
systemctl status watchlog.timer
systemctl status watchlog.service
```

`watchlog.service` имеет тип `oneshot` — он не "висит" постоянно в памяти, а запускается по срабатыванию таймера, выполняется и завершается со статусом `in`

`active (dead)`, что ожидаемо для oneshot-сервисов.

* * *

## 2\. spawn-fcgi.service

### 2.1. Установка пакетов

```bash
apt update
apt install spawn-fcgi php php-cgi php-cli apache2 libapache2-mod-fcgid -y
```

### 2.2. Конфигурация spawn-fcgi

```bash
mkdir -p /etc/spawn-fcgi
cat > /etc/spawn-fcgi/fcgi.conf << 'EOF'
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example:
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
EOF
```

### 2.3. Unit-файл spawn-fcgi.service

Основано на переработке классического init-скрипта (https://gist.github.com/cea2k/1318020) под systemd-формат:

```bash
cat > /etc/systemd/system/spawn-fcgi.service << 'EOF'
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

**Почему `Type=simple`, а не `forking`:** в конфиге используется флаг `-n` (no-fork, foreground mode) — spawn-fcgi не уходит в фон самостоятельно, а остаётся на переднем плане, поэтому systemd должен считать главным процессом именно этот, не форкнутый — отсюда `Type=simple`, а не `Type=forking`, как было бы для классического демона с PID-файлом.

### 2.4. Запуск и проверка

```bash
systemctl daemon-reload
systemctl start spawn-fcgi
systemctl status spawn-fcgi
```

```
root@otus:~# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Sun 2026-08-02 14:17:41 UTC; 8s ago
 Invocation: 5f40a5cd2d4d4d89bc96f35294a0891a
   Main PID: 12050 (php-cgi)
      Tasks: 33 (limit: 1888)
     Memory: 15.9M (peak: 16M)
        CPU: 31ms
     CGroup: /system.slice/spawn-fcgi.service
             ├─12050 /usr/bin/php-cgi
             ├─12055 /usr/bin/php-cgi
             ├─12056 /usr/bin/php-cgi
             ├─12057 /usr/bin/php-cgi
             ├─12058 /usr/bin/php-cgi
             ├─12059 /usr/bin/php-cgi
             ├─12060 /usr/bin/php-cgi
             ├─12061 /usr/bin/php-cgi
             ├─12062 /usr/bin/php-cgi
             ├─12063 /usr/bin/php-cgi
             ├─12064 /usr/bin/php-cgi
             ├─12065 /usr/bin/php-cgi
             ├─12066 /usr/bin/php-cgi
             ├─12067 /usr/bin/php-cgi
             ├─12068 /usr/bin/php-cgi
             ├─12069 /usr/bin/php-cgi
             ├─12070 /usr/bin/php-cgi
             ├─12071 /usr/bin/php-cgi
             ├─12072 /usr/bin/php-cgi
             ├─12073 /usr/bin/php-cgi
             ├─12074 /usr/bin/php-cgi
             ├─12075 /usr/bin/php-cgi
             ├─12076 /usr/bin/php-cgi
             ├─12077 /usr/bin/php-cgi
             ├─12078 /usr/bin/php-cgi
             ├─12079 /usr/bin/php-cgi
             ├─12080 /usr/bin/php-cgi
             ├─12081 /usr/bin/php-cgi
             ├─12082 /usr/bin/php-cgi
             ├─12083 /usr/bin/php-cgi
             ├─12084 /usr/bin/php-cgi
             ├─12085 /usr/bin/php-cgi
             └─12086 /usr/bin/php-cgi

Aug 02 14:17:41 otus systemd[1]: Started spawn-fcgi.service - Spawn-fcgi startup service by Otus.
root@otus:~#

```

* * *

## 3\. Nginx: несколько инстансов с разными конфигами

### 3.1. Установка Nginx

```bash
apt install nginx -y
```

### 3.2. Шаблонный unit-файл `nginx@.service`

Используется механизм systemd **template units** (`@`) — символ `%I` подставляет имя инстанса, указанное после `@` при запуске (`nginx@first`, `nginx@second`).

```bash
cat > /etc/systemd/system/nginx@.service << 'EOF'
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server (instance %I)
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOF
```

### 3.3. Конфигурационные файлы инстансов

Базовый `/etc/nginx/nginx.conf` копируется в два файла с изменением PID-файла и порта:

```bash
cp /etc/nginx/nginx.conf /etc/nginx/nginx-first.conf
cp /etc/nginx/nginx.conf /etc/nginx/nginx-second.conf
```

**`/etc/nginx/nginx-first.conf`** (ключевые изменения):

```nginx
cat > /etc/nginx/nginx-first.conf << 'EOF'
user www-data;
worker_processes auto;
pid /run/nginx-first.pid;
error_log /var/log/nginx/error-first.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access-first.log;

    gzip on;

    server {
        listen 9001;
        listen [::]:9001;
        server_name _;

        root /var/www/html;
        index index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
EOF
```

**`/etc/nginx/nginx-second.conf`** (ключевые изменения):

```nginx
cat > /etc/nginx/nginx-second.conf << 'EOF'
user www-data;
worker_processes auto;
pid /run/nginx-second.pid;
error_log /var/log/nginx/error-second.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    access_log /var/log/nginx/access-second.log;

    gzip on;

    server {
        listen 9002;
        listen [::]:9002;
        server_name _;

        root /var/www/html;
        index index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
EOF
```

> **Важно:** директива `include /etc/nginx/sites-enabled/*;` удалена в обоих файлах — иначе оба инстанса подхватят конфиг сайта по умолчанию с `listen 80`, что приведёт к конфликту портов между собой и со штатным `nginx.service` (если он тоже активен).

### 3.4. Отключение штатного nginx.service

```bash
systemctl stop nginx
systemctl disable nginx
```

### 3.5. Запуск инстансов

```bash
systemctl daemon-reload
systemctl start nginx@first
systemctl start nginx@second
systemctl status nginx@second
```

```
● nginx@second.service - A high performance web server and a reverse proxy server (instance second)
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled)
     Active: active (running) since Sun 2026-08-02 14:47:00 UTC; 6s ago
 Invocation: 2a3708cd13024a73a008ffbacde01036
       Docs: man:nginx(8)
    Process: 13577 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-second.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 13579 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 13580 (nginx)
      Tasks: 2 (limit: 1888)
     Memory: 2.4M (peak: 2.4M)
        CPU: 23ms
     CGroup: /system.slice/system-nginx.slice/nginx@second.service
             ├─13580 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;"
             └─13582 "nginx: worker process"

Aug 02 14:47:00 otus systemd[1]: Starting nginx@second.service - A high performance web server and a reverse proxy server (instance second)...
Aug 02 14:47:00 otus systemd[1]: Started nginx@second.service - A high performance web server and a reverse proxy server (instance second).

```

### 3.6. Проверка

Проверка слушаемых портов:

```bash
ss -tnulp | grep nginx
```

```
root@otus:~# ss -tnulp | grep nginx
tcp   LISTEN 0      511                              0.0.0.0:9002       0.0.0.0:*    users:(("nginx",pid=13582,fd=5),("nginx",pid=13580,fd=5))                 
tcp   LISTEN 0      511                              0.0.0.0:9001       0.0.0.0:*    users:(("nginx",pid=13565,fd=5),("nginx",pid=13563,fd=5))                 
tcp   LISTEN 0      511                                 [::]:9002          [::]:*    users:(("nginx",pid=13582,fd=6),("nginx",pid=13580,fd=6))                 
tcp   LISTEN 0      511                                 [::]:9001          [::]:*    users:(("nginx",pid=13565,fd=6),("nginx",pid=13563,fd=6))                 
```

Проверка процессов:

```bash
ps afx | grep nginx
```

```
root@otus:~# ps afx | grep nginx
  13635 pts/1    S+     0:00  |                       \_ grep --color=auto nginx
  13563 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  13565 ?        S      0:00  \_ nginx: worker process
  13580 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
  13582 ?        S      0:00  \_ nginx: worker process
root@otus:~#

```

Проверка ответа обоих инстансов:

```bash
root@otus:~# curl -I http://localhost:9001
HTTP/1.1 200 OK
Server: nginx/1.28.3
Date: Sun, 02 Aug 2026 14:49:11 GMT
Content-Type: text/html
Content-Length: 10672
Last-Modified: Sun, 02 Aug 2026 14:14:26 GMT
Connection: keep-alive
ETag: "6a6f50c2-29b0"
Accept-Ranges: bytes

root@otus:~# curl -I http://localhost:9002
HTTP/1.1 200 OK
Server: nginx/1.28.3
Date: Sun, 02 Aug 2026 14:49:20 GMT
Content-Type: text/html
Content-Length: 10672
Last-Modified: Sun, 02 Aug 2026 14:14:26 GMT
Connection: keep-alive
ETag: "6a6f50c2-29b0"
Accept-Ranges: bytes

root@otus:~#

```

Обе группы процессов Nginx запущены и слушают разные порты (9001 и 9002) с раздельными PID-файлами (`/run/nginx-first.pid`, `/run/nginx-second.pid`) 

Дополнительно можно включить автозапуск инстансов:

```bash
systemctl enable nginx@first
systemctl enable nginx@second
```

* * *

&nbsp;

&nbsp;