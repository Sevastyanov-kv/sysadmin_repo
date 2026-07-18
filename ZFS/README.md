# Домашнее задание: Практические навыки работы с ZFS

## Введение

ZFS(Zettabyte File System) — файловая система, созданная компанией Sun Microsystems в 2004-2005 годах для ОС Solaris. Эта файловая система поддерживает большие объёмы данных, объединяет в себе концепции файловой системы, RAID-массивов, менеджера логических дисков и принципы легковесных файловых систем.

ZFS продолжает активно развиваться. К примеру проект FreeNAS использует возможности ZFS для реализации ОС для управления SAN/NAS хранилищ.

Из-за лицензионных ограничений, поддержка ZFS в GNU/Linux ограничена. По умолчанию ZFS отсутствует в ядре linux.

Основное преимущество ZFS — это её полный контроль над физическими носителями, логическими томами и постоянное поддержание консистенции файловой системы. Оперируя на разных уровнях абстракции данных, ZFS способна обеспечить высокую скорость доступа к ним, контроль их целостности, а также минимизацию фрагментации данных. ZFS гибко настраивается, позволяет в процессе работы изменять объём дискового пространства и задавать разный размер блоков данных для разных применений, обеспечивает параллельность выполнения операций чтения-записи

## Описание домашнего задания

1.  **Определить алгоритм с наилучшим сжатием**
    - определить, какие алгоритмы сжатия поддерживает ZFS (gzip, zle, lzjb, lz4);
    - создать 4 файловые системы, на каждой применить свой алгоритм сжатия;
    - для сжатия использовать текстовый файл (или группу файлов).
2.  **Определить настройки пула**
    - командой `zpool import` собрать пул ZFS;
    - командами `zfs`/`zpool` определить: размер хранилища, тип pool, `recordsize`, тип сжатия, тип контрольной суммы.
3.  **Работа со снапшотами**
    - скопировать файл из удалённой директории (`zfs receive`);
    - восстановить файл локально;
    - найти зашифрованное сообщение в файле `secret_message`.

## Стенд

ВМ Ubuntu 26.04, системный диск `sda` (25G) + 8 дополнительных дисков по 512M (`sdb`–`sdi`):

```bash
kirills@otus:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda                         8:0    0   25G  0 disk
├─sda1                      8:1    0    1M  0 part
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part
  └─ubuntu--vg-ubuntu--lv 252:0    0 11.5G  0 lvm  /
sdb                         8:16   0  512M  0 disk
sdc                         8:32   0  512M  0 disk
sdd                         8:48   0  512M  0 disk
sde                         8:64   0  512M  0 disk
sdf                         8:80   0  512M  0 disk
sdg                         8:96   0  512M  0 disk
sdh                         8:112  0  512M  0 disk
sdi                         8:128  0  512M  0 disk
```

* * *

## Задание 1. Определение алгоритма с наилучшим сжатием

### 1.1. Установка утилит ZFS

```bash
sudo -i
apt update
apt install -y zfsutils-linux
```

> ZFS в GNU/Linux не входит в ядро по умолчанию из-за лицензионных ограничений (CDDL vs GPL), поэтому используется отдельный модуль `zfs.ko` из пакета `zfsutils-linux`.

**Вывод команды `zfs version`:**

```bash
root@otus:~# zfs version
zfs-2.4.1-1ubuntu5
zfs-kmod-2.4.1-1ubuntu5

```

### 1.2. Создание 4 пулов (зеркала из пар дисков)

```bash
zpool create otus1 mirror /dev/sdb /dev/sdc
zpool create otus2 mirror /dev/sdd /dev/sde
zpool create otus3 mirror /dev/sdf /dev/sdg
zpool create otus4 mirror /dev/sdh /dev/sdi
```

Проверка:

```bash
zpool list
```

```bash
root@otus:~# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   122K   480M        -         -     3%     0%  1.00x    ONLINE  -
otus2   480M   120K   480M        -         -     3%     0%  1.00x    ONLINE  -
otus3   480M   120K   480M        -         -     3%     0%  1.00x    ONLINE  -
otus4   480M   140K   480M        -         -     3%     0%  1.00x    ONLINE  -

```

```bash
zpool status
```

```bash
root@otus:~# zpool status
  pool: otus1
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors

```

### 1.3. Назначение разных алгоритмов сжатия

ZFS поддерживает следующие основные алгоритмы: `lzjb`, `gzip` (`gzip-1`…`gzip-9`), `zle`, `lz4` (в новых версиях также `zstd`).

```bash
zfs set compression=lzjb   otus1
zfs set compression=lz4    otus2
zfs set compression=gzip-9 otus3
zfs set compression=zle    otus4
```

Проверка, что у всех ФС разные алгоритмы:

```bash
zfs get all | grep compression
```

```
root@otus:~# zfs get all | grep compression
otus1  compression             lzjb                    local
otus2  compression             lz4                     local
otus3  compression             gzip-9                  local
otus4  compression             zle                     local

```

> Сжатие применяется только к файлам, записанным **после** включения настройки — уже существующие данные не пересжимаются.

### 1.4. Загрузка одинакового файла во все пулы

```bash
for i in {1..4}; do
  wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log
done
```

> Исходный файл имеет размер 40 мб.

```bash
ls -l /otus*
```

```bash
root@otus:~# ls -l /otus*
/otus1:
total 22130
-rw-r--r-- 1 root root 41250559 Jul  2 07:31 pg2600.converter.log

/otus2:
total 18022
-rw-r--r-- 1 root root 41250559 Jul  2 07:31 pg2600.converter.log

/otus3:
total 10972
-rw-r--r-- 1 root root 41250559 Jul  2 07:31 pg2600.converter.log

/otus4:
total 40315
-rw-r--r-- 1 root root 41250559 Jul  2 07:31 pg2600.converter.log

```

### 1.5. Сравнение степени сжатия

```bash
zfs list
```

```bash
root@otus:~# zfs list
NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1  21.7M   330M  21.6M  /otus1
otus2  17.7M   334M  17.6M  /otus2
otus3  10.8M   341M  10.7M  /otus3
otus4  39.5M   313M  39.4M  /otus4

```

```bash
zfs get all | grep compressratio | grep -v ref
```

```bash
root@otus:~# zfs get all | grep compressratio | grep -v ref
otus1  compressratio           1.82x                   -
otus2  compressratio           2.23x                   -
otus3  compressratio           3.66x                   -
otus4  compressratio           1.00x                   -

```

Наилучшее сжатие на использованном файле показывает алгоритм gzip-9

* * *

## Задание 2. Определение настроек пула

### 2.1. Подготовка тестового архива для импорта

```bash
cd ~
wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
tar -xzvf archive.tar.gz
```

```bash
root@otus:~# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb

```

### 2.2. Проверка возможности импорта

```bash
zpool import -d zpoolexport/
```

> - **`zpool import`** без указания конкретного имени пула работает в режиме сканирования: ищет на дисках/файлах метаданные ZFS-пулов и выводит информацию о найденных пулах (какие есть, в каком они состоянии, из чего состоят), но ничего не монтирует и не подключает.
> - **`-d zpoolexport/`** — указывает, где искать. По умолчанию `zpool import` сканирует все блочные устройства в `/dev`. Флаг `-d <каталог>` говорит: искать не среди дисков, а в указанной директории — в моем случае это распакованный архив с файлами `filea` и `fileb`, которые представляют собой файлы-образы дисков (ZFS умеет создавать vdev не только на реальных дисках, но и на обычных файлах).

```bash
root@otus:~# zpool import -d zpoolexport/
  pool: otus
    id: 6554193320433390805
 state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
config:

        otus                         ONLINE
          mirror-0                   ONLINE
            /root/zpoolexport/filea  ONLINE
            /root/zpoolexport/fileb  ONLINE

```

### 2.3. Импорт пула

```bash
zpool import -d zpoolexport/ otus
zpool status
```

```bash
root@otus:~# zpool import -d zpoolexport/ otus
root@otus:~# zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors

```

> Если пул с именем `otus` уже существует, можно импортировать под другим именем:  
> `zpool import -d zpoolexport/ otus newotus`

### 2.4. Определение параметров пула

```bash
zpool get all otus
```

```bash
root@otus:~# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.06M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  17%                            -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      4577142897028763526            -
otus  autotrim                       off                            default
otus  compatibility                  off                            default
otus  bcloneused                     0                              -
otus  bclonesaved                    0                              -
otus  bcloneratio                    1.00x                          -
otus  dedup_table_size               0                              -
otus  dedup_table_quota              auto                           default
otus  last_scrubbed_txg              0                              -
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
otus  feature@redaction_bookmarks    disabled                       local
otus  feature@redacted_datasets      disabled                       local
otus  feature@bookmark_written       disabled                       local
otus  feature@log_spacemap           disabled                       local
otus  feature@livelist               disabled                       local
otus  feature@device_rebuild         disabled                       local
otus  feature@zstd_compress          disabled                       local
otus  feature@draid                  disabled                       local
otus  feature@zilsaxattr             disabled                       local
otus  feature@head_errlog            disabled                       local
otus  feature@blake3                 disabled                       local
otus  feature@block_cloning          disabled                       local
otus  feature@vdev_zaps_v2           disabled                       local
otus  feature@redaction_list_spill   disabled                       local
otus  feature@raidz_expansion        disabled                       local
otus  feature@fast_dedup             disabled                       local
otus  feature@longname               disabled                       local
otus  feature@large_microzap         disabled                       local
otus  feature@dynamic_gang_header    disabled                       local
otus  feature@block_cloning_endian   disabled                       local
otus  feature@physical_rewrite       disabled                       local

```

```bash
zfs get all otus
```

```bash
root@otus:~# zfs get all otus
NAME  PROPERTY                VALUE                   SOURCE
otus  type                    filesystem              -
otus  creation                Fri May 15  4:00 2020   -
otus  used                    2.04M                   -
otus  available               350M                    -
otus  referenced              24K                     -
otus  compressratio           1.00x                   -
otus  mounted                 yes                     -
otus  quota                   none                    default
otus  reservation             none                    default
otus  recordsize              128K                    local
otus  mountpoint              /otus                   default
otus  sharenfs                off                     default
otus  checksum                sha256                  local
otus  compression             zle                     local
otus  atime                   on                      default
otus  devices                 on                      default
otus  exec                    on                      default
otus  setuid                  on                      default
otus  readonly                off                     default
otus  zoned                   off                     default
otus  snapdir                 hidden                  default
otus  aclmode                 discard                 default
otus  aclinherit              restricted              default
otus  createtxg               1                       -
otus  canmount                on                      default
otus  xattr                   sa                      default
otus  copies                  1                       default
otus  version                 5                       -
otus  utf8only                off                     -
otus  normalization           none                    -
otus  casesensitivity         sensitive               -
otus  vscan                   off                     default
otus  nbmand                  off                     default
otus  sharesmb                off                     default
otus  refquota                none                    default
otus  refreservation          none                    default
otus  guid                    14592242904030363272    -
otus  primarycache            all                     default
otus  secondarycache          all                     default
otus  usedbysnapshots         0B                      -
otus  usedbydataset           24K                     -
otus  usedbychildren          2.01M                   -
otus  usedbyrefreservation    0B                      -
otus  logbias                 latency                 default
otus  objsetid                54                      -
otus  dedup                   off                     default
otus  mlslabel                none                    default
otus  sync                    standard                default
otus  dnodesize               legacy                  default
otus  refcompressratio        1.00x                   -
otus  written                 24K                     -
otus  logicalused             1020K                   -
otus  logicalreferenced       12K                     -
otus  volmode                 default                 default
otus  filesystem_limit        none                    default
otus  snapshot_limit          none                    default
otus  filesystem_count        none                    default
otus  snapshot_count          none                    default
otus  snapdev                 hidden                  default
otus  acltype                 off                     default
otus  context                 none                    default
otus  fscontext               none                    default
otus  defcontext              none                    default
otus  rootcontext             none                    default
otus  relatime                on                      default
otus  redundant_metadata      all                     default
otus  overlay                 on                      default
otus  encryption              off                     default
otus  keylocation             none                    default
otus  keyformat               none                    default
otus  pbkdf2iters             0                       default
otus  special_small_blocks    0                       default
otus  prefetch                all                     default
otus  direct                  standard                default
otus  longname                off                     default
otus  defaultuserquota        0                       -
otus  defaultgroupquota       0                       -
otus  defaultprojectquota     0                       -
otus  defaultuserobjquota     0                       -
otus  defaultgroupobjquota    0                       -
otus  defaultprojectobjquota  0                       -

```

### 2.5. Точечные запросы по нужным параметрам

| Параметр | Команда |
| --- | --- |
| Размер хранилища | `zfs get available otus` |
| Тип пула (RO/RW) | `zfs get readonly otus` |
| `recordsize` | `zfs get recordsize otus` |
| Тип сжатия | `zfs get compression otus` |
| Контрольная сумма | `zfs get checksum otus` |

```bash
zfs get available otus
zfs get readonly otus
zfs get recordsize otus
zfs get compression otus
zfs get checksum otus
```

```bash
root@otus:~# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
root@otus:~# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
root@otus:~# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
root@otus:~# zfs get compression otus
NAME  PROPERTY     VALUE           SOURCE
otus  compression  zle             local
root@otus:~# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local

```

* * *

## Задание 3. Работа со снапшотом, поиск секретного сообщения

### 3.1. Загрузка файла снапшота

```bash
wget -O otus_task2.file --no-check-certificate 'https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download'

```

### 3.2. Восстановление файловой системы из снапшота

```bash
zfs receive otus/test@today < otus_task2.file
```

### 3.3. Поиск файла secret_message

```bash
find /otus/test -name "secret_message"
```

```bash
root@otus:~# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message

```

### 3.4. Просмотр содержимого

```bash
cat /otus/test/task1/file_mess/secret_message
```

```bash
root@otus:~# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/

```

* * *

## Источники

- [ZFS — Википедия](https://ru.wikipedia.org/wiki/ZFS)
- [Что такое ZFS? И почему люди от неё без ума? — Habr](https://habr.com/ru/post/424651/)
- [Oracle Solaris ZFS Administration Guide](https://docs.oracle.com/cd/E19253-01/819-5461/gbcya/index.html)
- [ZFS — xgu.ru](http://xgu.ru/wiki/ZFS)