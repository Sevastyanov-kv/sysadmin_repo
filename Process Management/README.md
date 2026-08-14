# Домашнее задание: Управление процессами

## Цель работы

Научиться анализировать состояние процессов в Linux и управлять их выполнением, используя данные виртуальной файловой системы `/proc`.

## Выбранный вариант

**Вариант 1. Реализация аналога `ps ax`**

Обоснование выбора: этот вариант наиболее полно демонстрирует базовый принцип работы утилит диагностики процессов в Linux — все они (`ps`, `top`, `htop` и т.д.) в конечном счёте читают информацию не из отдельной "базы процессов", а прямо из псевдо-файловой системы `/proc`, которую ядро генерирует "на лету".

* * *

## Теоретическая база

### Что такое `/proc`

`/proc` — это виртуальная (псевдо) файловая система. Она не хранит данные на диске, а представляет собой "окно" в структуры данных ядра. Файлы и каталоги внутри `/proc` создаются динамически при обращении к ним.

Для каждого работающего процесса ядро создаёт каталог `/proc/<PID>/`, где `<PID>` — идентификатор процесса. Внутри этого каталога лежат десятки файлов с информацией о процессе. Нас интересуют три:

| Файл | Что содержит |
| --- | --- |
| `/proc/<PID>/status` | Человекочитаемая сводка о процессе: имя, состояние, PPID, использование памяти, UID/GID и т.д. |
| `/proc/<PID>/comm` | Короткое имя команды (executable name), без пути и аргументов |
| `/proc/<PID>/cmdline` | Полная командная строка запуска (с аргументами, но байты разделены `\0`) |

### Поля, которые извлекает скрипт

- **PID** — Process ID, определяется по имени каталога `/proc/<PID>`.
- **PPID** (Parent PID) — берётся из строки `PPid:` файла `status`. Показывает, каким процессом был порождён (fork) данный процесс.
- **State (ST)** — берётся из строки `State:` файла `status`. Однобуквенный код состояния процесса:
    - `R` — running (выполняется или готов к выполнению)
    - `S` — sleeping (прерываемый сон, ждёт события)
    - `D` — uninterruptible sleep (обычно ожидание дискового I/O)
    - `T` — stopped (остановлен, например сигналом SIGSTOP)
    - `Z` — zombie (процесс завершился, но родитель ещё не считал его код возврата через `wait()`)
    - `I` — idle (простаивающий поток ядра)
- **CMD** — имя команды, читается из `/proc/<PID>/comm`.

### Почему именно `status`, а не парсинг `stat`

Файл `/proc/<PID>/stat` тоже содержит PPID и state, но в виде позиционных полей одной строки без подписей — это неудобно и опасно парсить (например, имя процесса в `stat` может содержать пробелы и скобки, что ломает разбиение по пробелу). Файл `status` даёт те же данные в виде `Ключ:\tЗначение`, что гораздо надёжнее для awk/grep.

* * *

## Исходный код скрипта

bash

```bash
#!/bin/bash
# myps.sh - простой аналог "ps ax"
# Собирает информацию о процессах напрямую из файловой системы /proc

printf "%-7s %-7s %-3s %-20s\n" "PID" "PPID" "ST" "CMD"

for pid_dir in /proc/[0-9]*; do
    pid=$(basename "$pid_dir")

    # Файл status содержит PPID и State в удобном текстовом виде
    [ -r "$pid_dir/status" ] || continue

    ppid=$(awk '/^PPid:/ {print $2}' "$pid_dir/status")
    state=$(awk '/^State:/ {print $2}' "$pid_dir/status")

    # comm - короткое имя команды (без аргументов)
    if [ -r "$pid_dir/comm" ]; then
        cmd=$(cat "$pid_dir/comm")
    else
        cmd="?"
    fi

    printf "%-7s %-7s %-3s %-20s\n" "$pid" "$ppid" "$state" "$cmd"
done
```

### Разбор кода построчно

1.  `printf "%-7s ..."` — печатает заголовок таблицы с выравниванием по левому краю (флаг `-` в формате `%-7s`).
2.  `for pid_dir in /proc/[0-9]*` — здесь ключевой момент: шаблон `[0-9]*` отбирает в `/proc` только те подкаталоги, имя которых начинается с цифры. Это отсекает служебные записи вроде `/proc/self`, `/proc/net`, `/proc/cpuinfo`, которые не являются PID-каталогами.
3.  `pid=$(basename "$pid_dir")` — из полного пути `/proc/1234` получаем просто `1234`.
4.  `[ -r "$pid_dir/status" ] || continue` — защита от гонки (race condition): пока скрипт перебирает список процессов, некоторые из них могут завершиться. Если файл `status` уже недоступен — пропускаем этот PID, чтобы `awk`/`cat` не падали с ошибкой.
5.  `awk '/^PPid:/ {print $2}'` — ищет строку, начинающуюся с `PPid:`, и выводит второе поле (само число).
6.  Аналогично для `State:` — там значение состоит из буквы и слова в скобках, например `S (sleeping)`, поэтому берём только `$2`, то есть саму букву-код.
7.  `cat "$pid_dir/comm"` — читает имя команды. `comm` всегда однострочный и без пробелов-разделителей аргументов, поэтому чтение простым `cat` безопасно.

* * *

## Проверка работы на системе

Команда запуска:

```bash
kirills@otus:/opt$ sudo ./myps.sh
```

Пример результата (фрагмент полного вывода, всего было получено 52 строки, включая заголовок):

```bash
kirills@otus:/opt$ sudo ./myps.sh
[sudo: authenticate] Password:
PID     PPID    ST  CMD
1       0       S   systemd
10      2       I   kworker/0:0H-kblockd
1013    1       S   nfsdcld
1017    2       I   kworker/0:4-cgroup_release
1084    1       S   systemd-network
11      2       I   kworker/0:1-cgroup_free
1114    2       I   kworker/R-cfg80211
12      2       I   kworker/u4:0-writeback
1241    1       S   chronyd-starter
1242    1       S   dbus-daemon
1247    1       S   fsidd
1250    1       S   networkd-dispat
1252    1       S   polkitd
1259    1       S   systemd-logind
1261    1       S   udisksd
13      2       I   kworker/R-mm_percpu_wq
1300    1       S   apache2
1306    1       S   cron
1314    1       S   rpc.idmapd
1333    1       S   rpc.mountd
1339    1       S   rpc.statd
1342    1241    S   chronyd
1357    1       S   unattended-upgr
1365    1342    S   chronyd
1373    1       S   rsyslogd
1377    1       S   agetty
1388    2       I   lockd
14      2       S   ksoftirqd/0
1403    1       S   ModemManager
1411    2       I   nfsd
1414    2       I   nfsd
1415    2       I   nfsd
1416    2       I   nfsd
1417    2       I   nfsd
1418    2       I   nfsd
1419    2       I   nfsd
1420    2       I   nfsd
1421    2       I   nfsd
1422    2       I   nfsd
1423    2       I   nfsd
1424    2       I   nfsd
1425    2       I   nfsd
1426    2       I   nfsd
1427    2       I   nfsd
1428    2       I   nfsd
1492    1300    S   apache2
1497    1300    S   apache2
1498    1300    S   apache2
15      2       I   rcu_preempt
16      2       S   rcu_exp_par_gp_kthread_worker/0
1631    1       S   sshd
1633    1631    S   sshd-session
1637    1631    S   sshd-session
1640    1       S   systemd
1643    1640    S   (sd-pam)
17      2       S   rcu_exp_gp_kthread_worker
18      2       S   migration/0
1847    1633    S   sshd-session
1848    1637    S   sshd-session
1849    1848    S   sftp-server
1850    1847    S   bash
19      2       S   kprobe-optimizer
1911    1850    S   sudo
1914    1911    S   sudo
1915    1914    S   myps.sh
2       0       S   kthreadd
20      2       S   idle_inject/0
21      2       S   cpuhp/0
22      2       S   kdevtmpfs
23      2       I   kworker/R-inet_frag_wq
24      2       I   rcu_tasks_kthread
25      2       I   rcu_tasks_rude_kthread
26      2       S   kauditd
27      2       S   khungtaskd
28      2       S   oom_reaper
29      2       I   kworker/u4:1-writeback
3       2       S   pool_workqueue_release
30      2       I   kworker/u4:2-writeback
31      2       I   kworker/R-writeback
32      2       S   kcompactd0
33      2       S   ksmd
34      2       S   khugepaged
35      2       I   kworker/R-kblockd
36      2       I   kworker/R-blkcg_punt_bio
37      2       I   kworker/R-kintegrityd
372     2       S   scsi_eh_2
373     2       I   kworker/R-scsi_tmf_2
38      2       S   irq/9-acpi
39      2       I   kworker/R-tpm_dev_wq
4       2       I   kworker/R-rcu_gp
40      2       I   kworker/R-ata_sff
41      2       I   kworker/R-md_bitmap
42      2       I   kworker/R-md_llbitmap_io
43      2       I   kworker/R-md_llbitmap_unplug
44      2       I   kworker/R-edac-poller
45      2       I   kworker/R-devfreq_wq
452     2       I   kworker/R-kdmflush/252:0
46      2       S   watchdogd
47      2       I   kworker/R-quota_events_unbound
48      2       S   kswapd0
49      2       S   ecryptfs-kthread
493     2       S   jbd2/dm-0-8
494     2       I   kworker/R-ext4-rsv-conversion
5       2       I   kworker/R-sync_wq
50      2       I   kworker/R-kthrotld
51      2       I   kworker/R-acpi_thermal_pm
52      2       S   scsi_eh_0
53      2       I   kworker/R-scsi_tmf_0
54      2       S   scsi_eh_1
55      2       I   kworker/R-scsi_tmf_1
56      2       I   kworker/u4:3-events_unbound
57      2       I   kworker/u4:4-ext4-rsv-conversion
58      2       I   kworker/u4:5
59      2       I   kworker/R-mld
6       2       I   kworker/R-kvfree_rcu_reclaim
60      2       I   kworker/R-ipv6_addrconf
61      2       I   kworker/0:2-events
62      2       I   kworker/0:1H
63      2       I   kworker/R-kstrp
65      2       I   kworker/u5:0
7       2       I   kworker/R-slub_flushwq
76      2       I   kworker/R-charger_manager
777     2       S   psimon
787     1       S   systemd-journal
8       2       I   kworker/R-netns
814     2       I   kworker/R-kmpathd
817     2       I   kworker/R-kmpath_handlerd
820     2       I   kworker/R-rpciod
821     2       I   kworker/R-xprtiod
825     1       S   multipathd
841     1       S   systemd-resolve
846     1       S   systemd-udevd
850     2       S   psimon
9       2       I   kworker/0:0-events
920     2       S   irq/18-vmwgfx
923     2       I   kworker/R-ttm
952     2       S   jbd2/sda2-8
953     2       I   kworker/R-ext4-rsv-conversion
996     1       S   rpcbind
997     2       I   kworker/0:3-cgroup_free
kirills@otus:/opt$

```

### Интерпретация результата

### 1\. Корень дерева процессов

```
1       0       S   systemd
2       0       S   kthreadd
```

- **`systemd`** (PID 1) — родитель всех пользовательских процессов и служб.
- **`kthreadd`** (PID 2) — родитель всех ядерных потоков.

Всё, что имеет PPID = 2 и имя вида `kworker/*`, `rcu_*`, `ksoftirqd/*`, `nfsd`, `jbd2/*` и т.п. — это потоки ядра, не пользовательские процессы. Их состояние почти всегда `I` (idle) или `S` (sleeping), потому что они пробуждаются только когда есть работа.

### 3\. Дерево пользовательских служб (PPID = 1)

Все процессы с PPID = 1 — это службы, запущенные напрямую через `systemd`:

```
1300    1       S   apache2
1631    1       S   sshd
1373    1       S   rsyslogd
1306    1       S   cron
825     1       S   multipathd
...
```

&nbsp;

* * *

## Вывод

Скрипт `myps.sh` подтверждает, что вся информация о процессах, которую показывают стандартные утилиты (`ps`, `top`), физически берётся из `/proc/<PID>/{status,comm,cmdline,...}`. Реализованы 4 обязательных поля: **PID**, **PPID**, **состояние процесса** и **имя команды**. Скрипт протестирован на работающей системе, вывод соответствует ожидаемому (иерархия `kthreadd → kworker/*` видна корректно).

```markdown
## Запуск

    chmod +x myps.sh
    ./myps.sh

## Требования
- bash
- awk
- доступ на чтение к /proc (обычно доступно любому пользователю без sudo)

## Пример
    ./myps.sh
```