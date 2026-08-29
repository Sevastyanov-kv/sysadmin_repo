# Домашнее задание: Практика с SELinux

## Цель домашнего задания

Диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений.

## Задачи

1.  Запустить Nginx на нестандартном порту тремя разными способами:
    - переключатели `setsebool`;
    - добавление нестандартного порта в имеющийся тип;
    - формирование и установка модуля SELinux.
2.  Обеспечить работоспособность приложения (DNS-сервер BIND) при включённом SELinux — стенд [vagrant_selinux_dns_problems](https://github.com/Nickmob/vagrant_selinux_dns_problems).

* * *

## Подготовка стенда

Стенд разворачивается через Vagrant + VirtualBox (репозиторий [vagrant_selinux](https://github.com/Nickmob/vagrant_selinux)):

```bash
vagrant up
vagrant ssh
sudo -i
```

После разворачивания на машине уже установлен Nginx, слушающий TCP-порт **4881** (проброшен на хост), и **включён SELinux**. Попытка запустить nginx завершается ошибкой:

```
Aug 26 18:25:19 selinux nginx[9075]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 26 18:25:19 selinux nginx[9075]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Aug 26 18:25:19 selinux nginx[9075]: nginx: configuration file /etc/nginx/nginx.conf test failed
Aug 26 18:25:19 selinux systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
```

Предварительные проверки:

```bash
[root@selinux nginx.service.d]# systemctl status firewalld    # firewalld отключен — не мешает
○ firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; preset: enabled)
     Active: inactive (dead)
       Docs: man:firewalld(1)
                   
[root@selinux nginx.service.d]# nginx -t                      # конфиг синтаксически корректен
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

[root@selinux nginx.service.d]# getenforce                     # Enforcing — SELinux блокирует запрещённую активность
Enforcing                 
```

Вывод: конфигурация и firewall не при чём — блокировку создаёт именно SELinux, поскольку порт 4881 не промаркирован типом, разрешённым для `httpd_t` (домен, в котором работает nginx).

* * *

## Часть 1. Запуск nginx на нестандартном порту тремя способами

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта

```
[root@selinux nginx.service.d]# grep 4881 /var/log/audit/audit.log
type=AVC msg=audit(1787768397.314:709): avc:  denied  { name_bind } for  pid=6331 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=AVC msg=audit(1787768719.016:763): avc:  denied  { name_bind } for  pid=9075 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
[root@selinux nginx.service.d]#
```

Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим

```bash
grep 1787768719.016:763 /var/log/audit/audit.log | audit2why
```

```
type=AVC msg=audit(1787768719.016:763): avc:  denied  { name_bind } for  pid=9075 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
[root@selinux nginx.service.d]#
```

Применяем рекомендацию и перезапускаем сервис:

```bash
setsebool -P nis_enabled on
systemctl restart nginx
systemctl status nginx

[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Thu 2026-08-27 06:40:10 UTC; 12s ago
    Process: 9297 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 9298 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 9299 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 9300 (nginx)
      Tasks: 3 (limit: 11997)
     Memory: 2.9M
        CPU: 27ms
     CGroup: /system.slice/nginx.service
             ├─9300 "nginx: master process /usr/sbin/nginx"
             ├─9301 "nginx: worker process"
             └─9302 "nginx: worker process"

Aug 27 06:40:10 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 27 06:40:10 selinux nginx[9298]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 27 06:40:10 selinux nginx[9298]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 27 06:40:10 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверка через браузер: `http://127.0.0.1:4881` — страница nginx открывается.

Проверка состояния булева параметра:

```bash
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```

Откатываем изменение (чтобы протестировать следующий способ «с нуля»):

```bash
setsebool -P nis_enabled off
```

### Способ 2 — добавление порта в существующий тип (semanage port)

Смотрим, какие типы уже существуют для http-трафика:

```bash
semanage port -l | grep http
```

```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Тип `http_port_t` — это ровно то, что разрешено домену `httpd_t`. Добавляем в него порт 4881:

```bash
semanage port -a -t http_port_t -p tcp 4881
semanage port -l | grep http_port_t

[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

```
http_port_t   tcp   4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```

```bash
systemctl restart nginx
systemctl status nginx      # active (running)

[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Thu 2026-08-27 07:25:42 UTC; 10s ago
    Process: 9345 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 9347 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 9348 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 9350 (nginx)
      Tasks: 3 (limit: 11997)
     Memory: 2.9M
        CPU: 25ms
     CGroup: /system.slice/nginx.service
             ├─9350 "nginx: master process /usr/sbin/nginx"
             ├─9351 "nginx: worker process"
             └─9352 "nginx: worker process"

Aug 27 07:25:42 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 27 07:25:42 selinux nginx[9347]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 27 07:25:42 selinux nginx[9347]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 27 07:25:42 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

Проверка в браузере: `http://127.0.0.1:4881` работает.

Удаление изменения (для демонстрации отката и перехода к способу 3):

```bash
semanage port -d -t http_port_t -p tcp 4881
semanage port -l | grep http_port_t   # 4881 больше нет
setsebool -P nis_enabled off
systemctl restart nginx               # снова падает: bind() ... Permission denied
```

### Способ 3 — формирование и установка модуля SELinux (audit2allow)

Пробуем снова запустить nginx (снова ошибка), затем анализируем свежие записи в аудите:

```bash
systemctl start nginx     # снова failed
grep nginx /var/log/audit/audit.log
```

```
type=SERVICE_START msg=audit(1787831186.810:84): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'UID="root" AUID="unset"

```

Генерируем модуль политики на основе логов и устанавливаем его:

```bash
grep nginx /var/log/audit/audit.log | audit2allow -M nginx
semodule -i nginx.pp
systemctl start nginx
systemctl status nginx     # active (running)

● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Thu 2026-08-27 11:52:17 UTC; 12s ago
    Process: 994 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 995 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 997 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 998 (nginx)
      Tasks: 3 (limit: 11997)
     Memory: 2.9M
        CPU: 34ms
     CGroup: /system.slice/nginx.service
             ├─ 998 "nginx: master process /usr/sbin/nginx"
             ├─ 999 "nginx: worker process"
             └─1000 "nginx: worker process"

Aug 27 11:52:17 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Aug 27 11:52:17 selinux nginx[995]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 27 11:52:17 selinux nginx[995]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 27 11:52:17 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

После установки модуля nginx запускается без ошибок

Просмотр установленных модулей и удаление созданного:

```bash
semodule -l | grep nginx
semodule -r nginx
```

### Сравнение трёх способов

| Способ | Команда | Область действия | Персистентность | Уместность |
| --- | --- | --- | --- | --- |
| `setsebool` | `setsebool -P nis_enabled on` | Глобальный булев переключатель, затрагивает не только nginx | Да (`-P`) | Низкая — включает доступ шире, чем нужно, использует несвязанный по смыслу булев |
| `semanage port` | `semanage port -a -t http_port_t -p tcp 4881` | Только указанный порт+протокол в рамках существующего, предназначенного для web типа | Да  | Высокая — целевое и семантически верное решение для смены порта http-сервиса |
| Модуль SELinux (`audit2allow`) | `audit2allow -M nginx && semodule -i nginx.pp` | Ровно то, что зафиксировано в логах аудита | Да  | Средняя — универсально, но требует ручной проверки сгенерированных правил |

* * *

## Часть 2. Диагностика проблемы динамического обновления DNS-зоны

### Разворачивание стенда

```bash
vagrant up
vagrant status

D:\Обучение\OTUS\15.SELinux - когда все запрещено\vagrant_selinux_dns_problems-main>vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)
```

### Воспроизведение проблемы

С клиента пытаемся обновить зону:

```bash
vagrant ssh client
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
```

```
update failed: SERVFAIL
```

### Диагностика

На клиенте `audit2why` ничего не показывает — проблема не здесь:

```bash
sudo -i
cat /var/log/audit/audit.log | audit2why   # пусто
```

Переходим на DNS-сервер `ns01` и смотрим аудит там:

```bash
vagrant ssh ns01
sudo -i
cat /var/log/audit/audit.log | audit2why
```

```
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1787907667.478:428): avc:  denied  { open } for  pid=2628 comm="20-chrony-dhcp" path="/etc/sysconfig/network-scripts/ifcfg-eth1" dev="sda4" ino=17270566 scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=unconfined_u:object_r:user_tmp_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1788003324.658:1629): avc:  denied  { write } for  pid=13935 comm="isc-net-0001" name="dynamic" dev="sda4" ino=51218433 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]#
```

В логах мы видим, что ошибка в контексте безопасности. Целевой контекст named_conf_t.

Для сравнения посмотрим существующую зону (localhost) и её контекст:

```bash
[root@ns01 ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 Aug 13 11:27 /var/named/named.localhost
[root@ns01 ~]#
```

Проверяем реальный каталог с конфигами зоны:

```
[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_conf_t:s0      121 Aug 28 09:03 .
drwxr-xr-x. 85 root root  system_u:object_r:etc_t:s0            8192 Aug 28 09:03 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_conf_t:s0   56 Aug 28 09:03 dynamic
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      784 Aug 28 09:03 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      610 Aug 28 09:03 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      609 Aug 28 09:03 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_conf_t:s0      657 Aug 28 09:03 named.newdns.lab
[root@ns01 ~]#
```

**Корень проблемы:** файлы зон физически лежат в `/etc/named`, но по политике SELinux этому каталогу должен соответствовать тип `named_conf_t` (конфигурация named), а не `named_zone_t` (файлы зон, в том числе с правом на запись демоном `named_t` в динамический раздел). Проверяем, что предписывает политика для файлов в разных каталогах:

```bash
sudo semanage fcontext -l | grep named
```

```
/etc/rndc.*        regular file  system_u:object_r:named_conf_t:s0
/var/named(/.*)?    all files     system_u:object_r:named_zone_t:s0
...
```

То есть зоны «правильно» хранить в `/var/named`, а не в `/etc/named` — в данном стенде это архитектурная особенность (нестандартное расположение), которая и приводит к конфликту с политикой SELinux.

**Изменим тип контекста безопасности для каталога /etc/named: sudo chcon -R -t named_zone_t /etc/named**

```bash
chcon -R -t named_zone_t /etc/named
ls -laZ /etc/named

drw-rwx---.  3 root named system_u:object_r:named_zone_t:s0      121 Aug 28 09:03 .
drwxr-xr-x. 85 root root  system_u:object_r:etc_t:s0            8192 Aug 28 09:03 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_zone_t:s0   56 Aug 28 09:03 dynamic
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      784 Aug 28 09:03 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      610 Aug 28 09:03 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      609 Aug 28 09:03 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      657 Aug 28 09:03 named.newdns.lab
[root@ns01 ~]#

```

Повторяем обновление зоны с клиента:

```bash
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit

dig www.ddns.lab
```

```
[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.16.23-RH <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48294
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 09e6a04f6971a828010000006a92ca8d6d7ca3a5fead189d (good)
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; Query time: 3 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Aug 29 12:03:25 UTC 2026
;; MSG SIZE  rcvd: 85

[vagrant@client ~]$
```

Изменение зоны прошло успешно, `dig` подтверждает наличие новой A-записи.

### Проверка после перезагрузки

После перезагрузки хостов запись в зоне сохраняется (`dig @192.168.50.10 www.ddns.lab` по-прежнему отдаёт `192.168.50.15`), но сам **контекст безопасности не является постоянным**: `chcon` не добавляет правило в базу semanage, поэтому при следующей перемаркировке (`restorecon`) контекст вернётся к тому, что прописан в политике:

```bash
restorecon -v -R /etc/named
```

```
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:named_conf_t:s0
restorecon reset /etc/named/dynamic context ...named_zone_t:s0->...named_conf_t:s0
...
```

Это наглядно демонстрирует разницу между «временным» изменением контекста (`chcon`) и «постоянным» правилом политики (`semanage fcontext`) 

* * *

## Глоссарий

- **AVC (Access Vector Cache)** — кэш решений SELinux о разрешении/запрете доступа; записи об отказах (`avc: denied`) попадают в `/var/log/audit/audit.log`.
- **`audit2why`** — утилита, объясняющая по записи в логе аудита причину блокировки и предлагающая команду для её устранения.
- **`audit2allow`** — утилита, генерирующая политику (модуль `.pp` или правило `.te`) на основе записей аудита, разрешающую зафиксированные обращения.
- **Домен (`_t` для процессов)** — тип контекста безопасности процесса, например `httpd_t`, `named_t`.
- **Тип файла/ресурса (`_t`)** — часть SELinux-контекста, определяющая, что это за объект (`http_port_t`, `named_zone_t`, `named_conf_t`).
- **`setsebool [-P]`** — включение/выключение булева переключателя политики; `-P` делает изменение постоянным.
- **`semanage port`** — управление привязкой портов к типам SELinux (какие домены могут биндиться на какой порт).
- **`semanage fcontext`** — управление правилами контекстов файловой системы, применяемыми при `restorecon`/relabel.
- **`chcon`** — разовая смена контекста файла/каталога без изменения политики; не переживает relabel.
- **`restorecon`** — восстановление контекстов файлов в соответствие с текущей политикой (базой `semanage fcontext`).
- **`semodule`** — установка (`-i`), удаление (`-r`) и просмотр (`-l`) загружаемых модулей политики SELinux.
- **`getenforce` / `setenforce`** — просмотр и смена режима SELinux (Enforcing / Permissive / Disabled).

* * *

## Вопросы для самопроверки

1.  Почему при добавлении порта 4881 через `semanage port -a -t http_port_t -p tcp 4881` не потребовалось перезагружать хост?
2.  В чём принципиальная разница между `chcon` и `semanage fcontext` с точки зрения поведения после `restorecon`?
3.  Почему `audit2why` не всегда предлагает архитектурно наилучшее решение (пример с `nis_enabled`)?
4.  Почему проблема с DNS-обновлением проявлялась в логах аудита именно на сервере `ns01`, а не на клиенте?
5.  Какой домен (не тип файла, а именно домен процесса) блокировался при попытке nginx забиндиться на порт 4881, и какой — при попытке `named` записать данные в каталог `dynamic`?
6.  Что произойдёт с доступом nginx к порту 4881, если одновременно применить и `setsebool -P nis_enabled on`, и позже `setsebool -P nis_enabled off`, при этом модуль из `audit2allow` остаётся установленным?
7.  Почему для сетевых портов используется `semanage port`, а не `semanage fcontext` (который относится к файловой системе)?