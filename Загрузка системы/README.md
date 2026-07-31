# Домашнее задание: Загрузка системы. Работа с загрузчиком

## Задание

1.  Включить отображение меню GRUB.
2.  Попасть в систему без пароля несколькими способами.
3.  Установить систему с LVM, после чего переименовать VG (Volume Group).

* * *

## 1\. Включение отображения меню GRUB

По умолчанию GRUB настроен так, чтобы не показывать меню и не делать задержку перед загрузкой — конфигурация находится в `/etc/default/grub`.

**Шаг 1.** Открываю конфиг загрузчика:

```bash
kirills@otus:~$ sudo nano /etc/default/grub
```

**Шаг 2.** Вношу два изменения:

- закомментировал строку, скрывающую меню:
    
    ```
    #GRUB_TIMEOUT_STYLE=hidden
    ```
    
- выставил задержку в 10 секунд для выбора пункта меню:
    
    ```
    GRUB_TIMEOUT=10
    ```
    

**Шаг 3.** Применяю конфигурацию и перезагружаюсь:

```bash
kirills@otus:~$ sudo update-grub
Sourcing file `/etc/default/grub'
Sourcing file `/etc/default/grub.d/kdump-tools.cfg'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-7.0.0-27-generic
Found initrd image: /boot/initrd.img-7.0.0-27-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done
kirills@otus:~$

kirills@otus:~$ sudo reboot
```

**Результат:** при следующей загрузке в окне виртуальной машины появляется меню GRUB с задержкой в 10 секунд.

> <img width="640" height="559" alt="image" src="https://github.com/user-attachments/assets/f0009076-f2b0-4d4f-a743-305a43fcd470" />


**Вывод:** параметр `GRUB_TIMEOUT_STYLE=hidden` отвечает именно за скрытие меню (не путать с `GRUB_TIMEOUT=0`, который отвечает только за отсутствие задержки при уже видимом меню). Оба параметра нужно скорректировать, чтобы меню было видно и было время его выбрать.

* * *

## 2\. Вход в систему без пароля

Для обоих способов виртуальную машину нужно перезагрузить и на экране меню GRUB нажать `e` для редактирования пункта загрузки.

> <img width="871" height="757" alt="image" src="https://github.com/user-attachments/assets/b9f4c993-ac2b-4f47-ac77-9f40662bb2cb" />


### Способ 1. `init=/bin/bash`

**Шаг 1.** В строке, начинающейся с `linux`, дописываю в конец `init=/bin/bash`.

**Шаг 2.** Нажимаю `Ctrl+X` (или F10) для загрузки с изменённым параметром.

Ядро вместо стандартного `init`/`systemd` запускает `/bin/bash` — я попадаю прямо в шелл от имени root, минуя аутентификацию.

**Нюанс:** корневая файловая система в этот момент смонтирована в режиме read-only, поэтому перед любыми изменениями (например, сбросом пароля) её нужно перемонтировать на запись:

```bash
root@otus:/# mount -o remount,rw /
```

Проверить результат можно так:

```bash
mount | grep root
```

### Способ 2. Recovery mode

**Шаг 1.** В меню GRUB на первом уровне выбираю пункт **Advanced options for Ubuntu**.

**Шаг 2.** Из списка ядер выбираю пункт с пометкой **(recovery mode)**.

Загружается меню восстановления (`recovery menu`).

> <img width="723" height="478" alt="image" src="https://github.com/user-attachments/assets/81451340-86c9-4644-bd91-3015db2b4c5c" />


**Шаг 3.** Сначала выбираю пункт **network** — это поднимает сеть и, что важно, перемонтирует корневую ФС в режим read/write.

**Шаг 4.** Затем выбираю пункт **root** — попадаю в консоль с правами root без ввода пароля.

В этой консоли уже можно свободно работать с системой: менять пароли, редактировать конфиги, чинить систему.

Оба способа эксплуатируют одну и ту же особенность GRUB — возможность на этапе загрузчика (до ввода пароля пользователя) передать ядру произвольные параметры или выбрать альтернативный режим загрузки. Отсюда практический вывод по безопасности: без пароля на сам GRUB (`GRUB_PASSWORD`/`superusers`) и без шифрования диска физический доступ к машине = полный доступ к системе.

* * *

## 3\. Установка системы с LVM и переименование VG

Система Ubuntu 26.04 установлена со стандартной разметкой диска с использованием LVM.

**Шаг 1.** Смотрю текущее состояние — список Volume Group:

```bash
kirills@otus:~$ sudo vgs
[sudo: authenticate] Password:
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n- <23.00g 11.50g

```

Интересует вторая строка — имя тома `ubuntu-vg`.

**Шаг 2.** Переименовываю VG:

```bash
kirills@otus:~$ sudo vgrename ubuntu-vg ubuntu-otus
  Volume group "ubuntu-vg" successfully renamed to "ubuntu-otus"
kirills@otus:~$

```

**Шаг 3.** Правлю `/boot/grub/grub.cfg` — везде заменяю старое имя VG на новое. Важный нюанс: в путях устройств `/dev/mapper/...` дефис внутри имени VG в файле экранируется и записывается как двойной дефис (`ubuntu--vg` → `ubuntu--otus`), это нужно учитывать при замене, иначе загрузчик не найдёт логический том.

<img width="1531" height="79" alt="image" src="https://github.com/user-attachments/assets/d322ca8b-43a9-4532-af23-0e28608d297c" />


После перезагрузки виртуальная машина не стартует:  
<img width="740" height="526" alt="image" src="https://github.com/user-attachments/assets/24add484-bd27-4d57-9a6e-57a5a35c9ad1" />


Захожу в систему через загрузчик первым способом `init=/bin/bash` и ввожу следующие команды:

```bash
vgscan
vgchange -ay
ls /dev/mapper/
exit
dracut -f --regenerate-all
update-grub
reboot
```

<img width="762" height="541" alt="image" src="https://github.com/user-attachments/assets/4c189bc2-5d1f-40b4-8351-f367197a5311" />


<img width="715" height="701" alt="image" src="https://github.com/user-attachments/assets/b3b16d62-ffdd-418c-8bbf-b60eb44cb458" />


**Шаг 4.** Перезагружаюсь и проверяю, что система успешно стартовала с новым именем VG:

<img width="795" height="88" alt="image" src="https://github.com/user-attachments/assets/0daded34-d82d-4dd4-8b2c-b5bcf060d014" />


Переименование прошло успешно, система загружается штатно.

**Дополнительно:** при желании таким же образом (командой `lvrename`) можно переименовать и Logical Volume внутри VG.

Переименование VG — операция на уровне метаданных LVM, но GRUB хранит путь к корневому разделу как строку в `grub.cfg`, поэтому она не подхватывается автоматически. После любого изменения имён VG/LV обязательно нужно синхронизировать `grub.cfg` (через `update-grub` или ручную правку), иначе система перестанет загружаться.
