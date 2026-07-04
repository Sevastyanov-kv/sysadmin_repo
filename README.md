# Домашнее задание: Обновление ядра системы

## Цель работы

Научиться вручную обновлять ядро в операционной системе Linux (на примере Ubuntu).

## Описание задания

1.  Развернуть и запустить ВМ с Ubuntu.
2.  Обновить ядро ОС на новейшую стабильную версию из mainline-репозитория.
3.  Логировать процесс и исправить возникающие ошибки.
4.  Оформить отчет в README-файле в GitHub-репозитории.

### Исходные данные

- **Среда виртуализации:** VirtualBox
- **Образ ОС:** `ubuntu-26.04-live-server-amd64`

* * *

## Ход выполнения работы

### Шаг 1. Проверка текущей версии ядра

Подключаемся к созданной виртуальной машине по SSH и смотрим текущую версию ядра:

uname -r

```
kirills@otus:~$ uname -r
7.0.0-27-generic
```

### Шаг 2. Подготовка и скачивание пакетов нового ядра

В домашней директории пользователя создаем рабочую папку `kernel` и переходим в нее:

```bash
mkdir kernel && cd kernel
```

При помощи утилиты `wget` скачиваем необходимые DEB-пакеты ядра из mainline-репозитория Ubuntu (ветка v7.1.2):

```
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-headers-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-headers-7.1.2-070102_7.1.2-070102.202606271039_all.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-image-unsigned-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-modules-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
```

Проверяем содержимое директории `/home/kirills/kernel`:

```
ls -la
```

**Вывод команды:**

```
kirills@otus:~/kernel$ ls -la
total 201812
drwxrwxr-x 2 kirills kirills      4096 Jul  4 09:16 .
drwxr-x--- 5 kirills kirills      4096 Jul  4 09:02 ..
-rw-rw-r-- 1 kirills kirills   4107804 Jun 27 11:23 linux-headers-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
-rw-rw-r-- 1 kirills kirills  14861728 Jun 27 11:24 linux-headers-7.1.2-070102_7.1.2-070102.202606271039_all.deb
-rw-rw-r-- 1 kirills kirills  17490112 Jun 27 11:22 linux-image-unsigned-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
-rw-rw-r-- 1 kirills kirills 170178752 Jun 27 11:23 linux-modules-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
```

### Шаг 3. Установка пакетов ядра

Устанавливаем все скачанные пакеты одновременно с помощью `dpkg`:

```
sudo dpkg -i *.deb
```

**Вывод команды:**

```
kirills@otus:~/kernel$ sudo dpkg -i *.deb
Selecting previously unselected package linux-headers-7.1.2-070102.
(Reading database… 98708 files and directories currently installed.)
Preparing to unpack linux-headers-7.1.2-070102_7.1.2-070102.202606271039_all.deb…
Unpacking linux-headers-7.1.2-070102 (7.1.2-070102.202606271039)...
Selecting previously unselected package linux-headers-7.1.2-070102-generic.
Preparing to unpack linux-headers-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb…
Unpacking linux-headers-7.1.2-070102-generic (7.1.2-070102.202606271039)...
Selecting previously unselected package linux-image-unsigned-7.1.2-070102-generic.
Preparing to unpack linux-image-unsigned-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb…
Unpacking linux-image-unsigned-7.1.2-070102-generic (7.1.2-070102.202606271039)...
Selecting previously unselected package linux-modules-7.1.2-070102-generic.
Preparing to unpack linux-modules-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb…
Unpacking linux-modules-7.1.2-070102-generic (7.1.2-070102.202606271039)...
Setting up linux-headers-7.1.2-070102 (7.1.2-070102.202606271039)...
Setting up linux-headers-7.1.2-070102-generic (7.1.2-070102.202606271039)...
Setting up linux-modules-7.1.2-070102-generic (7.1.2-070102.202606271039)...
Setting up linux-image-unsigned-7.1.2-070102-generic (7.1.2-070102.202606271039)...
I: /boot/vmlinuz is now a symlink to vmlinuz-7.1.2-070102-generic
I: /boot/initrd.img is now a symlink to initrd.img-7.1.2-070102-generic
Processing triggers for linux-image-unsigned-7.1.2-070102-generic (7.1.2-070102.202606271039)...
```

### Шаг 4. Контроль файлов в /boot и обновление загрузчика

Проверяем появление новых компонентов ядра в директории `/boot`:

```
ls -al /boot
```

```
kirills@otus:~/kernel$ ls -al /boot
total 120668
drwxr-xr-x  4 root root     4096 Jul  4 09:43 .
drwxr-xr-x 20 root root     4096 Jul  3 14:28 ..
-rw-------  1 root root 11010417 Jun 18 16:54 System.map-7.0.0-27-generic
-rw-------  1 root root 11899340 Jun 27 10:39 System.map-7.1.2-070102-generic
-rw-r--r--  1 root root   308501 Jun 18 16:54 config-7.0.0-27-generic
-rw-r--r--  1 root root   307299 Jun 27 10:39 config-7.1.2-070102-generic
drwxr-xr-x  5 root root     4096 Jul  4 09:01 grub
lrwxrwxrwx  1 root root       31 Jul  4 09:43 initrd.img -> initrd.img-7.1.2-070102-generic
-rw-------  1 root root 65248457 Jul  3 14:30 initrd.img-7.0.0-27-generic
lrwxrwxrwx  1 root root       27 Jul  4 09:01 initrd.img.old -> initrd.img-7.0.0-27-generic
drwx------  2 root root    16384 Jul  3 14:25 lost+found
lrwxrwxrwx  1 root root       28 Jul  4 09:43 vmlinuz -> vmlinuz-7.1.2-070102-generic
-rw-------  1 root root 17283464 Jun 18 17:19 vmlinuz-7.0.0-27-generic
-rw-------  1 root root 17457664 Jun 27 10:39 vmlinuz-7.1.2-070102-generic
lrwxrwxrwx  1 root root       24 Jul  4 09:01 vmlinuz.old -> vmlinuz-7.0.0-27-generic
```

Обновляем конфигурацию загрузчика GRUB:

```
sudo update-grub
```

```
kirills@otus:~/kernel$ sudo update-grub
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/kdump-tools.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-7.1.2-070102-generic
Found linux image: /boot/vmlinuz-7.0.0-27-generic
Found initrd image: /boot/initrd.img-7.0.0-27-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
```

Выбираем загрузку нового ядра по умолчанию и отправляем ВМ в перезагрузку:

```
sudo grub-set-default 0
sudo reboot
```

## Возникшая проблема и её решение

**Ошибка при перезагрузке:** `VFS: Unable to mount root fs on unknown-block(0,0)` **Причина:** При установке mainline-ядра для него автоматически не сгенерировался файл начальной файловой системы RAM (`initrd.img-7.1.2-070102-generic`). Загрузчик выдал ошибку, так как при поиске образов (см. вывод `update-grub` выше) для нового ядра `vmlinuz-7.1.2` отсутствовал соответствующий `initrd`.

<img width="734" height="417" alt="Снимок экрана 2026-07-04 125201" src="https://github.com/user-attachments/assets/16f948d1-82ec-4a3b-84ab-6bb6838bb00a" />


### Решение проблемы:

1.  Перезагружаем ВМ и в меню GRUB (раздел "Advanced options") принудительно выбираем старое рабочее ядро `7.0.0-27-generic`.
    
2.  После успешной загрузки генерируем недостающий образ `initramfs` вручную:
    
    ```
    sudo update-initramfs -c -k 7.1.2-070102-generic
    ```
    
3.  Проверяем успешность создания файла в `/boot`:
    
    ```
    ls -al /boot
    ```
    
    Файл `initrd.img-7.1.2-070102-generic` успешно создан
    
4.  Повторно обновляем конфигурацию GRUB, чтобы загрузчик зафиксировал новый initrd-образ:
    
    ```
    sudo update-grub
    ```
    
    **Вывод команды:**
    
    ```
    Generating grub configuration file ...
    Found linux image: /boot/vmlinuz-7.1.2-070102-generic
    Found initrd image: /boot/initrd.img-7.1.2-070102-generic
    Found linux image: /boot/vmlinuz-7.0.0-27-generic
    Found initrd image: /boot/initrd.img-7.0.0-27-generic
    done
    ```
    
5.  Снова отправляем виртуальную машину в перезагрузку:
    
    ```
    sudo reboot
    ```
    

## Результат

После перезагрузки проверяем версию ядра системы:

```
uname -r
```

**Вывод:**

```
kirills@otus:~$ uname -r
7.1.2-070102-generic
```

Ядро системы успешно обновлено до версии `7.1.2-070102-generic`.
