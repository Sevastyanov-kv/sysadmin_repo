# ДЗ 26. Резервное копирование (BorgBackup, Vagrant + Ansible)

## 1\. Что такое BorgBackup

**BorgBackup (borg)** — консольная утилита резервного копирования с дедупликацией, сжатием и шифрованием. Написана на Python (с критичными частями на Cython/C), распространяется под лицензией BSD, работает на Linux, macOS и \*BSD. Является форком Attic.

Ключевые возможности:

| Возможность | Как реализована |
| --- | --- |
| **Дедупликация** | Данные режутся на чанки (chunker с изменяемой границей по содержимому). Одинаковые чанки хранятся один раз — даже между разными архивами и файлами. Поэтому второй и последующие бэкапы почти ничего не весят: сохраняются только изменившиеся чанки. |
| **Сжатие** | lz4 (быстро), zstd, zlib, lzma (сильно, но медленно); настраивается ключом `--compression`. |
| **Шифрование** | Аутентифицированное шифрование на стороне клиента (AES-256-CTR + HMAC-SHA256 или BLAKE2b). Сервер хранит только зашифрованные данные и не знает пароль. |
| **Удалённые репозитории** | По SSH: на сервере запускается `borg serve`. Доступ можно ограничить через `authorized_keys` (`command=...`, `--restrict-to-path`, `--append-only`). |
| **Удобное восстановление** | `borg extract` — достать файлы, `borg mount` — смонтировать архив через FUSE и ходить по нему как по обычному каталогу. |
| **Политика хранения** | `borg prune` удаляет старые архивы по правилам (`--keep-daily`, `--keep-monthly`, ...). |
| **Проверка целостности** | `borg check` проверяет репозиторий и архивы. |

Основные понятия:

- **Репозиторий (repository)** — хранилище данных (каталог на диске/удалённом сервере). Создаётся командой `borg init`.
- **Архив (archive)** — один снимок (бэкап) внутри репозитория; имеет имя (у нас — с датой и временем).
- **Режимы шифрования репозитория**: `none`, `authenticated`, `repokey` (ключ хранится в репозитории и защищён паролем), `keyfile` (ключ хранится на клиенте, в `~/.config/borg/keys`), а также варианты `*-blake2`.

Базовый цикл работы:

```bash
borg init --encryption=repokey-blake2 REPO     # создать репозиторий
borg create --stats REPO::имя_архива /путь     # сделать бэкап
borg list REPO                                 # список архивов
borg list REPO::имя_архива                     # список файлов в архиве
borg extract REPO::имя_архива путь/в/архиве    # восстановить
borg prune --keep-daily 7 REPO                 # удалить старые архивы по правилам
```

* * *

## 2\. Задание

Настроить стенд Vagrant с двумя виртуальными машинами: `backup_server` и `client`.

Настроить удалённый бэкап каталога `/etc` с сервера `client` при помощи borgbackup. Резервные копии должны соответствовать критериям:

1.  директория для резервных копий `/var/backup` — **отдельная точка монтирования** (для демонстрации достаточно 2 GB);
2.  репозиторий **зашифрован** ключом или паролем — на усмотрение студента;
3.  имя бэкапа содержит **время снятия** бэкапа;
4.  глубина бэкапа — **год**: хранить по последней копии на конец месяца, кроме последних трёх; последние три месяца — копии **на каждый день**. Должна быть правильно настроена политика удаления старых бэкапов;
5.  резервная копия снимается **каждые 5 минут** (частый запуск — для демонстрации);
6.  написан **скрипт** для снятия копий, запускается из cron-джобы либо systemd timer;
7.  настроено **логирование**: можно весь вывод перенаправлять в `logger` с тегом; если логи не в syslog — обязательна ротация.

Что сдать: Vagrantfile + Ansible, логи процесса бэкапа, описание процесса восстановления.

Порядок проверки:

- запустить стенд на 30 минут и убедиться, что копии снимаются;
- остановить бэкап, удалить (или переместить) `/etc` и восстановить его из бэкапа.

* * *

## 3\. Схема сети

```
                 Host (Vagrant + VirtualBox + Ansible)
                              |
              private network 192.168.11.0/24
        ┌─────────────────────┴─────────────────────┐
        |                                           |
 ┌──────┴────────────┐   SSH (borg serve)   ┌───────┴────────────┐
 │ client            │ ───────────────────► │ backup_server      │
 │ 192.168.11.150    │   borg create/prune  │ 192.168.11.160     │
 │ Ubuntu 22.04      │                      │ Ubuntu 22.04       │
 │                   │                      │                    │
 │ /etc  (источник)  │                      │ /dev/sdc (2 GB)    │
 │ borg-backup.timer │                      │   └─► /var/backup  │
 │ (каждые 5 минут)  │                      │ user: borg         │
 └───────────────────┘                      └────────────────────┘
```

| Хост | IP  | Роль |
| --- | --- | --- |
| `backup_server` | 192.168.11.160 | хранит репозиторий в `/var/backup` (отдельный диск 2 GB) |
| `client` | 192.168.11.150 | источник данных (`/etc`), запускает бэкап по таймеру |

* * *

## 4\. Особенности проектирования и реализации

| Решение | Почему |
| --- | --- |
| Репозиторий в `/var/backup/repo`, а не в корне `/var/backup` | В корне свежесозданной ext4 есть каталог `lost+found`, а `borg init` не создаёт репозиторий в непустом каталоге (ошибка `There is already something at /var/backup`). Точка монтирования остаётся `/var/backup`. |
| **ansible_local** вместо `ansible` | Хост — Windows, а Ansible не поддерживает Windows как управляющую машину. Ansible ставится внутрь каждой ВМ (`apt install ansible`) и применяет плейбук сам к себе; плейбук берётся из синхронизируемой папки `/vagrant`. |
| Обмен SSH-ключом через `/vagrant/keys` | Машины провижинятся независимо друг от друга, поэтому `backup_server` генерирует ключ и кладёт его в общую папку, а `client` забирает его оттуда (с правами 0600 на своей ФС). Каталог `keys/` в `.gitignore`. |
| **systemd timer** вместо cron | Удобно смотреть состояние (`systemctl list-timers`), логи попадают в journald, у сервиса есть зависимость от сети. |
| Таймер `OnActiveSec=1min` + `OnUnitActiveSec=5min` | Только `OnUnitActiveSec` не сработает, пока сервис ни разу не запускался (в методичке этот нюанс не учтён). `OnActiveSec` даёт первый запуск через минуту после старта таймера, дальше — каждые 5 минут после предыдущего запуска. |
| Отдельный диск 2 GB, ext4, монтирование по **UUID** в `/etc/fstab` | Требование «отдельная точка монтирования»; имя `/dev/sdX` может поменяться, UUID — нет. |
| Шифрование `repokey-blake2` + пароль | Ключ лежит в репозитории, защищён паролем. Пароль хранится на клиенте в `/etc/borg/borg.env` с правами `0600`, а не в unit-файле (unit-файлы читаемы всеми). В бою пароль нужно убирать в `ansible-vault`. |
| Отдельный SSH-ключ ed25519 клиента, на сервере — ограничение в `authorized_keys`: `command="borg serve --restrict-to-path /var/backup",restrict` | Даже при компрометации ключа клиента им нельзя получить shell на сервере и выйти за пределы `/var/backup`. |
| Имя архива `etc-{now:%Y-%m-%d_%H:%M:%S}` | Время снятия копии прямо в имени (требование п. 3). |
| Политика `prune`: `--keep-daily 90 --keep-monthly 9` | 90 дней — по копии на каждый день за последние ~3 месяца; ещё 9 месяцев — по последней копии месяца. Итого глубина ≈ 12 месяцев (см. пояснение ниже). |
| Логирование: скрипт → `logger -t borg-backup` → rsyslog → `/var/log/borg-backup.log` | Выполняется требование про `logger` с тегом; дополнительный файл удобен для просмотра, для него настроен `logrotate`. |

### Пояснение по retention

Правила `borg prune` применяются в порядке: `--keep-within`, `--keep-hourly`, `--keep-daily`, `--keep-weekly`, `--keep-monthly`, `--keep-yearly`. Каждое правило оставляет **последний** архив в своём периоде. Архивы, уже оставленные предыдущим правилом, повторно **не учитываются** в счётчике следующего. Поэтому:

- `--keep-daily 90` — по одному (последнему) архиву за каждый из последних 90 дней, в которых были бэкапы;
- `--keep-monthly 9` — ещё 9 «месячных» архивов **за пределами** уже сохранённого дневного окна → 3 месяца + 9 месяцев = **год**.

### Важный нюанс про 5-минутные копии и prune

Из-за `--keep-daily` внутри одного дня остаётся **только последняя** копия — это правильное поведение по условию задачи. Но при демонстрации «каждые 5 минут» в `borg list` вы увидите лишь одну копию за сегодня, а остальные будут удалены при очередном `prune`.

Чтобы на демонстрации были видны все копии за 30 минут, в скрипт добавлено **временное** правило `--keep-within {{ borg_keep_within }}` (по умолчанию `1h`). Это только для демонстрации; для «боевой» политики из условия задачи значение нужно сделать пустым — тогда ключ в команду не добавляется.

* * *

## 5\. Структура репозитория

```
.
├── README.md
├── Vagrantfile
├── .gitignore / .gitattributes
├── keys/                     # создаётся при запуске: SSH-ключ borg (в git не попадает)
└── ansible
    ├── playbook.yml
    ├── group_vars
    │   └── all.yml
    └── templates
        ├── borg.env.j2
        ├── borg-backup.sh.j2
        ├── borg-backup.service.j2
        ├── borg-backup.timer.j2
        ├── rsyslog-borg.conf.j2
        └── logrotate-borg.j2
```

* * *

## 6\. Vagrantfile

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_check_update = false

  # Хост — Windows: Ansible на нём не работает, поэтому используем ansible_local —
  # Ansible ставится ВНУТРИ каждой ВМ и применяет плейбук к самой себе.
  # Плейбук берётся из синхронизируемой папки /vagrant (корень проекта).
  # 1) ставим ansible из репозитория Ubuntu (пакет "ansible" включает нужные коллекции)
  config.vm.provision "install-ansible", type: "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive
    command -v ansible-playbook >/dev/null || (apt-get update -qq && apt-get install -y -qq ansible)
  SHELL

  # 2) запускаем плейбук; на каждой ВМ выполняются только «её» play-блоки
  config.vm.provision "borg-playbook", type: "ansible_local" do |ansible|
    ansible.install  = false
    ansible.playbook = "ansible/playbook.yml"
    ansible.compatibility_mode = "2.0"
    ansible.extra_vars = { ansible_python_interpreter: "/usr/bin/python3" }
  end

  # ---------- Сервер бэкапов ----------
  config.vm.define "backup_server" do |srv|
    srv.vm.hostname = "backup-server"
    srv.vm.network "private_network", ip: "192.168.11.160"

    srv.vm.provider "virtualbox" do |vb|
      vb.name   = "backup_server"
      vb.memory = 1024
      vb.cpus   = 1

      # Дополнительный диск 2 GB для /var/backup
      disk = File.join(File.dirname(__FILE__), ".vagrant", "backup_disk.vdi")
      unless File.exist?(disk)
        vb.customize ["createhd", "--filename", disk, "--size", 2048]
      end
      # В боксе ubuntu/jammy64 контроллер называется "SCSI";
      # порты 0 и 1 заняты системным диском и cloud-init диском
      vb.customize ["storageattach", :id,
                    "--storagectl", "SCSI",
                    "--port", 2, "--device", 0,
                    "--type", "hdd", "--medium", disk]
    end
  end

  # ---------- Клиент ----------
  config.vm.define "client" do |cl|
    cl.vm.hostname = "client"
    cl.vm.network "private_network", ip: "192.168.11.150"

    cl.vm.provider "virtualbox" do |vb|
      vb.name   = "client"
      vb.memory = 1024
      vb.cpus   = 1
    end

  end
end
```

> Примечания:
> 
> - `createhd` работает в VirtualBox 6.1 (по методичке) и в 7.x как устаревший алиас; в 7.x можно заменить на `["createmedium", "disk", "--filename", disk, "--size", 2048]`.
> - Файл диска лежит в `.vagrant/`. После `vagrant destroy` его нужно удалить вручную (`rm .vagrant/backup_disk.vdi`), если нужен чистый диск.

* * *

## 7\. Ansible

Ansible запускается **внутри ВМ** (`ansible_local`), на хосте (в т. ч. Windows) его устанавливать не нужно. Коллекции `ansible.posix` и `community.general` входят в пакет `ansible` из репозитория Ubuntu 22.04.

### ansible/group_vars/all.yml

```yaml
---
# --- Сервер бэкапов ---
borg_server_ip: 192.168.11.160
borg_user: borg
borg_repo_path: /var/backup
backup_disk: /dev/sdc            # второй диск (проверить: lsblk)

# --- Клиент ---
# Репозиторий — в подкаталоге: в корне только что созданной ext4 лежит lost+found,
# а borg init отказывается создавать репозиторий в непустом каталоге
borg_repo_dir: "{{ borg_repo_path }}/repo"
borg_repo: "ssh://{{ borg_user }}@{{ borg_server_ip }}{{ borg_repo_dir }}"
borg_passphrase: "Otus1234"      # для продакшена — ansible-vault!
borg_ssh_key: /root/.ssh/borg_ed25519
borg_backup_target: /etc
borg_archive_prefix: "etc-"
borg_tag: borg-backup

# --- Политика хранения ---
borg_keep_daily: 90              # последние ~3 месяца — по копии на каждый день
borg_keep_monthly: 9             # ещё 9 месяцев — по последней копии месяца (итого год)
borg_keep_within: "1H"           # ТОЛЬКО для демонстрации 5-минутных копий; пусто = выключено
```

### ansible/playbook.yml

```yaml
---
# Playbook запускается через ansible_local на КАЖДОЙ ВМ отдельно (inventory = только эта ВМ),
# поэтому play-блоки с hosts: backup_server / client выполняются каждый на своей машине.
# Порядок: backup_server (создаёт ключ и кладёт его в /vagrant/keys) -> client (забирает ключ).

# ============ 1. Общее: установка borgbackup ============
- name: Установка borgbackup на все хосты
  hosts: all
  become: true
  tasks:
    - name: Установить borgbackup
      ansible.builtin.apt:
        name: borgbackup
        state: present
        update_cache: true
        cache_valid_time: 3600

# ============ 2. Сервер: пользователь, диск, /var/backup, ключ ============
- name: Сервер бэкапов
  hosts: backup_server
  become: true
  tasks:
    - name: Создать пользователя borg
      ansible.builtin.user:
        name: "{{ borg_user }}"
        shell: /bin/bash
        create_home: true

    - name: Проверить, что дополнительный диск существует
      ansible.builtin.stat:
        path: "{{ backup_disk }}"
      register: disk_stat

    - name: Остановиться, если диска нет
      ansible.builtin.assert:
        that: disk_stat.stat.exists and disk_stat.stat.isblk
        fail_msg: "Диск {{ backup_disk }} не найден — проверьте lsblk и переменную backup_disk"

    - name: Создать файловую систему ext4 на диске
      community.general.filesystem:
        fstype: ext4
        dev: "{{ backup_disk }}"

    - name: Получить UUID файловой системы
      ansible.builtin.command: blkid -s UUID -o value {{ backup_disk }}
      register: disk_uuid
      changed_when: false

    - name: Создать точку монтирования
      ansible.builtin.file:
        path: "{{ borg_repo_path }}"
        state: directory

    - name: Смонтировать диск (запись в /etc/fstab по UUID)
      ansible.posix.mount:
        path: "{{ borg_repo_path }}"
        src: "UUID={{ disk_uuid.stdout }}"
        fstype: ext4
        opts: defaults
        state: mounted

    - name: Выдать права пользователю borg на /var/backup
      ansible.builtin.file:
        path: "{{ borg_repo_path }}"
        state: directory
        owner: "{{ borg_user }}"
        group: "{{ borg_user }}"
        mode: "0750"

    # --- SSH-ключ для клиента: создаётся на сервере, клиент заберёт его из /vagrant/keys ---
    - name: Каталог /root/.ssh
      ansible.builtin.file:
        path: /root/.ssh
        state: directory
        mode: "0700"

    - name: Сгенерировать SSH-ключ borg (если ещё нет)
      ansible.builtin.command:
        cmd: ssh-keygen -t ed25519 -N "" -C "borg@client" -f {{ borg_ssh_key }}
        creates: "{{ borg_ssh_key }}"

    - name: Положить пару ключей в общую папку Vagrant (/vagrant/keys)
      ansible.builtin.shell: |
        mkdir -p /vagrant/keys
        cp -f {{ borg_ssh_key }} {{ borg_ssh_key }}.pub /vagrant/keys/
      args:
        creates: /vagrant/keys/{{ borg_ssh_key | basename }}

    - name: Прочитать публичный ключ
      ansible.builtin.slurp:
        src: "{{ borg_ssh_key }}.pub"
      register: borg_pubkey_raw

    - name: Добавить ключ в authorized_keys пользователя borg с ограничениями
      ansible.posix.authorized_key:
        user: "{{ borg_user }}"
        key: "{{ borg_pubkey_raw.content | b64decode | trim }}"
        key_options: 'command="borg serve --restrict-to-path {{ borg_repo_path }}",restrict'

# ============ 3. Клиент: ключ, репозиторий, скрипт, таймер, логи ============
- name: Клиент — настройка бэкапа
  hosts: client
  become: true
  handlers:
    - name: Перезапустить rsyslog
      ansible.builtin.systemd:
        name: rsyslog
        state: restarted

  tasks:
    - name: Проверить, что сервер уже выложил ключ
      ansible.builtin.stat:
        path: /vagrant/keys/{{ borg_ssh_key | basename }}
      register: shared_key

    - name: Остановиться, если ключа нет
      ansible.builtin.assert:
        that: shared_key.stat.exists
        fail_msg: "Нет /vagrant/keys/{{ borg_ssh_key | basename }} — сначала должен отработать provisioning backup_server (vagrant provision backup_server)"

    - name: Каталог /root/.ssh
      ansible.builtin.file:
        path: /root/.ssh
        state: directory
        mode: "0700"

    - name: Установить приватный ключ borg из общей папки
      ansible.builtin.copy:
        src: /vagrant/keys/{{ borg_ssh_key | basename }}
        dest: "{{ borg_ssh_key }}"
        remote_src: true
        mode: "0600"

    - name: Каталог /etc/borg
      ansible.builtin.file:
        path: /etc/borg
        state: directory
        mode: "0700"

    - name: Файл окружения (пароль, репозиторий, ssh-команда)
      ansible.builtin.template:
        src: templates/borg.env.j2
        dest: /etc/borg/borg.env
        mode: "0600"

    - name: Инициализировать репозиторий (если ещё не создан)
      ansible.builtin.shell: |
        set -a; . /etc/borg/borg.env; set +a
        if borg info "$BORG_REPO" >/dev/null 2>&1; then
          echo EXISTS
        else
          borg init --encryption=repokey-blake2 "$BORG_REPO" && echo CREATED
        fi
      args:
        executable: /bin/bash
      register: borg_init
      changed_when: "'CREATED' in borg_init.stdout"

    - name: Скрипт бэкапа
      ansible.builtin.template:
        src: templates/borg-backup.sh.j2
        dest: /usr/local/bin/borg-backup.sh
        mode: "0750"

    - name: systemd service
      ansible.builtin.template:
        src: templates/borg-backup.service.j2
        dest: /etc/systemd/system/borg-backup.service
        mode: "0644"

    - name: systemd timer
      ansible.builtin.template:
        src: templates/borg-backup.timer.j2
        dest: /etc/systemd/system/borg-backup.timer
        mode: "0644"

    - name: Правило rsyslog (лог в отдельный файл)
      ansible.builtin.template:
        src: templates/rsyslog-borg.conf.j2
        dest: /etc/rsyslog.d/30-borg-backup.conf
        mode: "0644"
      notify: Перезапустить rsyslog

    - name: Ротация лога
      ansible.builtin.template:
        src: templates/logrotate-borg.j2
        dest: /etc/logrotate.d/borg-backup
        mode: "0644"

    - name: Включить и запустить таймер
      ansible.builtin.systemd:
        name: borg-backup.timer
        enabled: true
        state: started
        daemon_reload: true
```

### ansible/templates/borg.env.j2

```ini
# Managed by Ansible
BORG_REPO={{ borg_repo }}
BORG_PASSPHRASE={{ borg_passphrase }}
BORG_RSH="ssh -i {{ borg_ssh_key }} -o StrictHostKeyChecking=accept-new -o ServerAliveInterval=10 -o ServerAliveCountMax=30"
```

### ansible/templates/borg-backup.sh.j2

```bash
#!/usr/bin/env bash
# Managed by Ansible
# Скрипт резервного копирования {{ borg_backup_target }} на удалённый borg-репозиторий.
# Переменные BORG_REPO, BORG_PASSPHRASE, BORG_RSH приходят из EnvironmentFile сервиса.

set -uo pipefail

TAG="{{ borg_tag }}"
TARGET="{{ borg_backup_target }}"
PREFIX="{{ borg_archive_prefix }}"

{
  rc=0

  echo "==== START backup ${TARGET} -> ${BORG_REPO} ===="

  # 1. Создание архива; в имени — время снятия копии
  borg create \
      --stats --show-rc \
      --compression lz4 \
      "${BORG_REPO}::${PREFIX}{now:%Y-%m-%d_%H:%M:%S}" \
      "${TARGET}" || rc=$?

  # 2. Очистка старых копий по политике хранения
  echo "---- prune ----"
  borg prune \
      --stats --show-rc \
      --prefix "${PREFIX}" \
{% if borg_keep_within %}
      --keep-within {{ borg_keep_within }} \
{% endif %}
      --keep-daily {{ borg_keep_daily }} \
      --keep-monthly {{ borg_keep_monthly }} \
      "${BORG_REPO}" || rc=$?

  # 3. Лёгкая проверка: только последний архив
  echo "---- check ----"
  borg check --archives-only --last 1 --show-rc "${BORG_REPO}" || rc=$?

  echo "==== END backup, итоговый код: ${rc} ===="
  exit "${rc}"
} 2>&1 | logger -t "${TAG}"

exit "${PIPESTATUS[0]}"
```

### ansible/templates/borg-backup.service.j2

```ini
# Managed by Ansible
[Unit]
Description=Borg backup of {{ borg_backup_target }} to {{ borg_server_ip }}
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
EnvironmentFile=/etc/borg/borg.env
ExecStart=/usr/local/bin/borg-backup.sh
Nice=10
```

### ansible/templates/borg-backup.timer.j2

```ini
# Managed by Ansible
[Unit]
Description=Run borg-backup every 5 minutes

[Timer]
# первый запуск — через минуту после старта таймера
OnActiveSec=1min
# далее — каждые 5 минут после предыдущего запуска сервиса
OnUnitActiveSec=5min
AccuracySec=1s
Unit=borg-backup.service

[Install]
WantedBy=timers.target
```

### ansible/templates/rsyslog-borg.conf.j2

```
# Managed by Ansible
# Сообщения с тегом borg-backup (из logger) — в отдельный файл
if $programname == '{{ borg_tag }}' then /var/log/borg-backup.log
```

### ansible/templates/logrotate-borg.j2

```
# Managed by Ansible
/var/log/borg-backup.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

* * *

## 8\. Запуск стенда

```bash
git clone <ссылка на ваш репозиторий>
cd <каталог>
vagrant up            # поднимет обе ВМ и выполнит ansible
vagrant status
```

Повторный прогон только Ansible (порядок важен: сначала `backup_server`, потом `client`):

```bash
vagrant provision
# или по отдельности
vagrant provision backup_server
vagrant provision client
```

Заходим на машины:

```bash
vagrant ssh client
vagrant ssh backup_server
```

* * *

## 9\. Проверка работы (стенд — 30 минут)

### 9.1. Сервер: отдельная точка монтирования

```bash
vagrant ssh backup_server
lsblk
findmnt /var/backup
df -h /var/backup
grep backup /etc/fstab
```

```bash
vagrant@backup-server:~$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0    74M  1 loop /snap/core22/2437
loop1    7:1    0  50.1M  1 loop /snap/snapd/27710
loop2    7:2    0 115.3M  1 loop /snap/lxd/40575
sda      8:0    0    40G  0 disk
└─sda1   8:1    0    40G  0 part /
sdb      8:16   0    10M  0 disk
sdc      8:32   0     2G  0 disk /var/backup
vagrant@backup-server:~$ findmnt /var/backup
TARGET      SOURCE   FSTYPE OPTIONS
/var/backup /dev/sdc ext4   rw,relatime
vagrant@backup-server:~$ df -h /var/backup
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc        2.0G  1.3M  1.8G   1% /var/backup
vagrant@backup-server:~$ grep backup /etc/fstab
UUID=cd30a2a6-2d7c-4a24-8217-a9decf83f357 /var/backup ext4 defaults 0 0
vagrant@backup-server:~$
```

Ожидаем: диск ~2 GB смонтирован в `/var/backup`, в `fstab` запись с `UUID=...`.

### 9.2. Клиент: таймер и сервис

```bash
vagrant ssh client
sudo -i
systemctl list-timers --all | grep borg
systemctl status borg-backup.timer
systemctl status borg-backup.service
```

```bash
vagrant@client:~$ sudo -i
root@client:~# systemctl list-timers --all | grep borg
Thu 2026-10-15 08:18:59 UTC 3min 47s left     Thu 2026-10-15 08:13:59 UTC 1min 12s ago borg-backup.timer              borg-backup.service
root@client:~# systemctl status borg-backup.timer
● borg-backup.timer - Run borg-backup every 5 minutes
     Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since Thu 2026-10-15 06:37:39 UTC; 1h 37min ago
    Trigger: Thu 2026-10-15 08:18:59 UTC; 3min 34s left
   Triggers: ● borg-backup.service

Oct 15 06:37:39 client systemd[1]: Started Run borg-backup every 5 minutes.
root@client:~# systemctl status borg-backup.service
○ borg-backup.service - Borg backup of /etc to 192.168.11.160
     Loaded: loaded (/etc/systemd/system/borg-backup.service; static)
     Active: inactive (dead) since Thu 2026-10-15 08:14:01 UTC; 1min 38s ago
TriggeredBy: ● borg-backup.timer
    Process: 14382 ExecStart=/usr/local/bin/borg-backup.sh (code=exited, status=0/SUCCESS)
   Main PID: 14382 (code=exited, status=0/SUCCESS)
        CPU: 867ms

Oct 15 08:14:00 client borg-backup[14384]:
Oct 15 08:14:00 client borg-backup[14384]:                        Unique chunks         Total chunks
Oct 15 08:14:00 client borg-backup[14384]: Chunk index:                     760                10140
Oct 15 08:14:00 client borg-backup[14384]: ------------------------------------------------------------------------------
Oct 15 08:14:00 client borg-backup[14384]: terminating with success status, rc 0
Oct 15 08:14:00 client borg-backup[14384]: ---- check ----
Oct 15 08:14:01 client borg-backup[14384]: terminating with success status, rc 0
Oct 15 08:14:01 client borg-backup[14384]: ==== END backup, итоговый код: 0 ====
Oct 15 08:14:01 client systemd[1]: borg-backup.service: Deactivated successfully.
Oct 15 08:14:01 client systemd[1]: Finished Borg backup of /etc to 192.168.11.160.
root@client:~#
```

Ожидаем: таймер `active (waiting)`, следующий запуск в пределах 5 минут, сервис после запусков — `inactive (dead)` с `status=0/SUCCESS`.

### 9.3. Клиент: список копий (через ~30 минут)

```bash
borg list "$BORG_REPO"
borg info "$BORG_REPO"
```

```bash
root@client:~# borg list "$BORG_REPO"
etc-2026-10-15_06:38:40              Thu, 2026-10-15 06:38:41 [576fb3f215178f7c14b6ec7a32ab8043f215ed91999b778e665496f5af3a928e]
etc-2026-10-15_07:18:48              Thu, 2026-10-15 07:18:48 [09d986a5222299e54b4c3f82679174041a692436351f8bb1bed6b6eb903abb02]
etc-2026-10-15_07:23:49              Thu, 2026-10-15 07:23:49 [1e0ae2d67ecfefdbbf834cef631350d326d79657cef910174efc0684097c1fb0]
etc-2026-10-15_07:28:50              Thu, 2026-10-15 07:28:51 [cff160664a582b17c7e25ddd4ed29cc44804d4d53a48b88a99d3b3a05c9f180e]
etc-2026-10-15_07:33:51              Thu, 2026-10-15 07:33:52 [9897a030328d1cb1d34ddc8235bacf6f578beab4897595ab7fb2867557085a7f]
etc-2026-10-15_07:38:52              Thu, 2026-10-15 07:38:53 [5847a630e8c8ba6a8d48ad1096de2eaaade816e45c572c89aa72f35d64c57a1b]
etc-2026-10-15_07:43:53              Thu, 2026-10-15 07:43:53 [ba7f5004a34d11d47ba38d876c8464b1360b7878dd27716981688dc8b73ab144]
etc-2026-10-15_07:48:54              Thu, 2026-10-15 07:48:54 [3679cfea0862b8c1864d4e330786593ff96b6a83dea866f1d83f119e2566a3dc]
etc-2026-10-15_07:53:55              Thu, 2026-10-15 07:53:56 [da0d8eda85ee8390745a69c0d1d09b1a2101ed8fa5e5dc132ada74488db12d2e]
etc-2026-10-15_07:58:56              Thu, 2026-10-15 07:58:56 [7e7f7ec3ee868aaf26e2b245b6685c634e7a59d7c997efbc07643d59dda2be09]
etc-2026-10-15_08:03:57              Thu, 2026-10-15 08:03:58 [596d4fe269904bf565aa2d4e7a0dd5e266ebb38c560397128a979ef089a45f0b]
etc-2026-10-15_08:08:58              Thu, 2026-10-15 08:08:58 [15a301e7b3e1d479d5126219c89be72658b313bfe3041e62b9a8a93d1736eb80]
etc-2026-10-15_08:13:59              Thu, 2026-10-15 08:14:00 [6d06a89afc1a28a38daf26b4736d6e508b60f01f41dd32202d601a92e8671002]
root@client:~# date
Thu Oct 15 08:18:26 UTC 2026
root@client:~# borg info "$BORG_REPO"
Repository ID: 52aff6ffc35b5263af47cdfe9504977788d56964949f938430382a6cd74b8bcf
Location: ssh://borg@192.168.11.160/var/backup/repo
Encrypted: Yes (repokey BLAKE2b)
Cache: /root/.cache/borg/52aff6ffc35b5263af47cdfe9504977788d56964949f938430382a6cd74b8bcf
Security dir: /root/.config/borg/security/52aff6ffc35b5263af47cdfe9504977788d56964949f938430382a6cd74b8bcf
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
All archives:               28.83 MB             12.79 MB              1.04 MB

                       Unique chunks         Total chunks
Chunk index:                     760                10140
root@client:~#
```

Ожидаем: архивы вида `etc-ГГГГ-ММ-ДД_ЧЧ:ММ:СС` с интервалом ~5 минут (за последний час, благодаря `--keep-within 1h`), репозиторий с `Encrypted: Yes (repokey BLAKE2b)`.

### 9.4. Проверка шифрования

Репозиторий без пароля не открывается:

```bash
env -u BORG_PASSPHRASE BORG_RSH="$BORG_RSH" BORG_REPO="$BORG_REPO" borg list </dev/null
```

```bash
root@client:~# env -u BORG_PASSPHRASE BORG_RSH="$BORG_RSH" BORG_REPO="$BORG_REPO" borg list </dev/null
Enter passphrase for key ssh://borg@192.168.11.160/var/backup/repo:
```

### 9.5. Логи

```bash
# файл лога (rsyslog)
tail -n 60 /var/log/borg-backup.log

# syslog / journald
grep borg-backup /var/log/syslog | tail -n 30
journalctl -u borg-backup.service --since "30 min ago" | tail -n 40
journalctl -t borg-backup --since "30 min ago" | tail -n 40

# ротация
cat /etc/logrotate.d/borg-backup
logrotate -d /etc/logrotate.d/borg-backup
```

```bash
tail -n 60 /var/log/borg-backup.log
Oct 15 08:19:00 client borg-backup: ==== START backup /etc -> ssh://borg@192.168.11.160/var/backup/repo ====
Oct 15 08:19:01 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:19:01 client borg-backup: Repository: ssh://borg@192.168.11.160/var/backup/repo
Oct 15 08:19:01 client borg-backup: Archive name: etc-2026-10-15_08:19:00
Oct 15 08:19:01 client borg-backup: Archive fingerprint: c84b3d8e6b2718d6249b71f62f7d0368b3d77c94736e140c8156cef1a99aad27
Oct 15 08:19:01 client borg-backup: Time (start): Thu, 2026-10-15 08:19:00
Oct 15 08:19:01 client borg-backup: Time (end):   Thu, 2026-10-15 08:19:01
Oct 15 08:19:01 client borg-backup: Duration: 0.07 seconds
Oct 15 08:19:01 client borg-backup: Number of files: 789
Oct 15 08:19:01 client borg-backup: Utilization of max. archive size: 0%
Oct 15 08:19:01 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:19:01 client borg-backup:                        Original size      Compressed size    Deduplicated size
Oct 15 08:19:01 client borg-backup: This archive:                2.22 MB            984.59 kB                644 B
Oct 15 08:19:01 client borg-backup: All archives:               31.05 MB             13.78 MB              1.04 MB
Oct 15 08:19:01 client borg-backup:
Oct 15 08:19:01 client borg-backup:                        Unique chunks         Total chunks
Oct 15 08:19:01 client borg-backup: Chunk index:                     761                10920
Oct 15 08:19:01 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:19:01 client borg-backup: terminating with success status, rc 0
Oct 15 08:19:01 client borg-backup: ---- prune ----
Oct 15 08:19:01 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:19:01 client borg-backup:                        Original size      Compressed size    Deduplicated size
Oct 15 08:19:01 client borg-backup: Deleted data:               -2.48 MB             -1.06 MB               -644 B
Oct 15 08:19:01 client borg-backup: All archives:               28.83 MB             12.79 MB              1.04 MB
Oct 15 08:19:01 client borg-backup:
Oct 15 08:19:01 client borg-backup:                        Unique chunks         Total chunks
Oct 15 08:19:01 client borg-backup: Chunk index:                     760                10140
Oct 15 08:19:01 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:19:01 client borg-backup: terminating with success status, rc 0
Oct 15 08:19:01 client borg-backup: ---- check ----
Oct 15 08:19:02 client borg-backup: terminating with success status, rc 0
Oct 15 08:19:02 client borg-backup: ==== END backup, итоговый код: 0 ====

root@client:~# grep borg-backup /var/log/syslog | tail -n 30
Oct 15 08:29:03 client borg-backup: Archive name: etc-2026-10-15_08:29:02
Oct 15 08:29:03 client borg-backup: Archive fingerprint: abca14a1805b9cf708adf5804b01b43b4559b563e895bf22bf99cc8016b26964
Oct 15 08:29:03 client borg-backup: Time (start): Thu, 2026-10-15 08:29:02
Oct 15 08:29:03 client borg-backup: Time (end):   Thu, 2026-10-15 08:29:03
Oct 15 08:29:03 client borg-backup: Duration: 0.07 seconds
Oct 15 08:29:03 client borg-backup: Number of files: 789
Oct 15 08:29:03 client borg-backup: Utilization of max. archive size: 0%
Oct 15 08:29:03 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup:                        Original size      Compressed size    Deduplicated size
Oct 15 08:29:03 client borg-backup: This archive:                2.22 MB            984.59 kB                646 B
Oct 15 08:29:03 client borg-backup: All archives:               31.05 MB             13.78 MB              1.04 MB
Oct 15 08:29:03 client borg-backup:
Oct 15 08:29:03 client borg-backup:                        Unique chunks         Total chunks
Oct 15 08:29:03 client borg-backup: Chunk index:                     761                10920
Oct 15 08:29:03 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup: terminating with success status, rc 0
Oct 15 08:29:03 client borg-backup: ---- prune ----
Oct 15 08:29:03 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup:                        Original size      Compressed size    Deduplicated size
Oct 15 08:29:03 client borg-backup: Deleted data:               -2.48 MB             -1.06 MB               -642 B
Oct 15 08:29:03 client borg-backup: All archives:               28.83 MB             12.79 MB              1.04 MB
Oct 15 08:29:03 client borg-backup:
Oct 15 08:29:03 client borg-backup:                        Unique chunks         Total chunks
Oct 15 08:29:03 client borg-backup: Chunk index:                     760                10140
Oct 15 08:29:03 client borg-backup: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup: terminating with success status, rc 0
Oct 15 08:29:03 client borg-backup: ---- check ----
Oct 15 08:29:04 client borg-backup: terminating with success status, rc 0
Oct 15 08:29:04 client borg-backup: ==== END backup, итоговый код: 0 ====
Oct 15 08:29:04 client systemd[1]: borg-backup.service: Deactivated successfully.
root@client:~#

root@client:~# journalctl -u borg-backup.service --since "30 min ago" | tail -n 40
Oct 15 08:24:02 client borg-backup[14422]: ---- check ----
Oct 15 08:24:03 client borg-backup[14422]: terminating with success status, rc 0
Oct 15 08:24:03 client borg-backup[14422]: ==== END backup, итоговый код: 0 ====
Oct 15 08:24:03 client systemd[1]: borg-backup.service: Deactivated successfully.
Oct 15 08:24:03 client systemd[1]: Finished Borg backup of /etc to 192.168.11.160.
Oct 15 08:29:02 client systemd[1]: Starting Borg backup of /etc to 192.168.11.160...
Oct 15 08:29:02 client borg-backup[14433]: ==== START backup /etc -> ssh://borg@192.168.11.160/var/backup/repo ====
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]: Repository: ssh://borg@192.168.11.160/var/backup/repo
Oct 15 08:29:03 client borg-backup[14433]: Archive name: etc-2026-10-15_08:29:02
Oct 15 08:29:03 client borg-backup[14433]: Archive fingerprint: abca14a1805b9cf708adf5804b01b43b4559b563e895bf22bf99cc8016b26964
Oct 15 08:29:03 client borg-backup[14433]: Time (start): Thu, 2026-10-15 08:29:02
Oct 15 08:29:03 client borg-backup[14433]: Time (end):   Thu, 2026-10-15 08:29:03
Oct 15 08:29:03 client borg-backup[14433]: Duration: 0.07 seconds
Oct 15 08:29:03 client borg-backup[14433]: Number of files: 789
Oct 15 08:29:03 client borg-backup[14433]: Utilization of max. archive size: 0%
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]:                        Original size      Compressed size    Deduplicated size
Oct 15 08:29:03 client borg-backup[14433]: This archive:                2.22 MB            984.59 kB                646 B
Oct 15 08:29:03 client borg-backup[14433]: All archives:               31.05 MB             13.78 MB              1.04 MB
Oct 15 08:29:03 client borg-backup[14433]:
Oct 15 08:29:03 client borg-backup[14433]:                        Unique chunks         Total chunks
Oct 15 08:29:03 client borg-backup[14433]: Chunk index:                     761                10920
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]: terminating with success status, rc 0
Oct 15 08:29:03 client borg-backup[14433]: ---- prune ----
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]:                        Original size      Compressed size    Deduplicated size
Oct 15 08:29:03 client borg-backup[14433]: Deleted data:               -2.48 MB             -1.06 MB               -642 B
Oct 15 08:29:03 client borg-backup[14433]: All archives:               28.83 MB             12.79 MB              1.04 MB
Oct 15 08:29:03 client borg-backup[14433]:
Oct 15 08:29:03 client borg-backup[14433]:                        Unique chunks         Total chunks
Oct 15 08:29:03 client borg-backup[14433]: Chunk index:                     760                10140
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]: terminating with success status, rc 0
Oct 15 08:29:03 client borg-backup[14433]: ---- check ----
Oct 15 08:29:04 client borg-backup[14433]: terminating with success status, rc 0
Oct 15 08:29:04 client borg-backup[14433]: ==== END backup, итоговый код: 0 ====
Oct 15 08:29:04 client systemd[1]: borg-backup.service: Deactivated successfully.
Oct 15 08:29:04 client systemd[1]: Finished Borg backup of /etc to 192.168.11.160.
root@client:~#

root@client:~# journalctl -t borg-backup --since "30 min ago" | tail -n 40
Oct 15 08:24:02 client borg-backup[14422]:
Oct 15 08:24:02 client borg-backup[14422]:                        Unique chunks         Total chunks
Oct 15 08:24:02 client borg-backup[14422]: Chunk index:                     760                10140
Oct 15 08:24:02 client borg-backup[14422]: ------------------------------------------------------------------------------
Oct 15 08:24:02 client borg-backup[14422]: terminating with success status, rc 0
Oct 15 08:24:02 client borg-backup[14422]: ---- check ----
Oct 15 08:24:03 client borg-backup[14422]: terminating with success status, rc 0
Oct 15 08:24:03 client borg-backup[14422]: ==== END backup, итоговый код: 0 ====
Oct 15 08:29:02 client borg-backup[14433]: ==== START backup /etc -> ssh://borg@192.168.11.160/var/backup/repo ====
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]: Repository: ssh://borg@192.168.11.160/var/backup/repo
Oct 15 08:29:03 client borg-backup[14433]: Archive name: etc-2026-10-15_08:29:02
Oct 15 08:29:03 client borg-backup[14433]: Archive fingerprint: abca14a1805b9cf708adf5804b01b43b4559b563e895bf22bf99cc8016b26964
Oct 15 08:29:03 client borg-backup[14433]: Time (start): Thu, 2026-10-15 08:29:02
Oct 15 08:29:03 client borg-backup[14433]: Time (end):   Thu, 2026-10-15 08:29:03
Oct 15 08:29:03 client borg-backup[14433]: Duration: 0.07 seconds
Oct 15 08:29:03 client borg-backup[14433]: Number of files: 789
Oct 15 08:29:03 client borg-backup[14433]: Utilization of max. archive size: 0%
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]:                        Original size      Compressed size    Deduplicated size
Oct 15 08:29:03 client borg-backup[14433]: This archive:                2.22 MB            984.59 kB                646 B
Oct 15 08:29:03 client borg-backup[14433]: All archives:               31.05 MB             13.78 MB              1.04 MB
Oct 15 08:29:03 client borg-backup[14433]:
Oct 15 08:29:03 client borg-backup[14433]:                        Unique chunks         Total chunks
Oct 15 08:29:03 client borg-backup[14433]: Chunk index:                     761                10920
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]: terminating with success status, rc 0
Oct 15 08:29:03 client borg-backup[14433]: ---- prune ----
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]:                        Original size      Compressed size    Deduplicated size
Oct 15 08:29:03 client borg-backup[14433]: Deleted data:               -2.48 MB             -1.06 MB               -642 B
Oct 15 08:29:03 client borg-backup[14433]: All archives:               28.83 MB             12.79 MB              1.04 MB
Oct 15 08:29:03 client borg-backup[14433]:
Oct 15 08:29:03 client borg-backup[14433]:                        Unique chunks         Total chunks
Oct 15 08:29:03 client borg-backup[14433]: Chunk index:                     760                10140
Oct 15 08:29:03 client borg-backup[14433]: ------------------------------------------------------------------------------
Oct 15 08:29:03 client borg-backup[14433]: terminating with success status, rc 0
Oct 15 08:29:03 client borg-backup[14433]: ---- check ----
Oct 15 08:29:04 client borg-backup[14433]: terminating with success status, rc 0
Oct 15 08:29:04 client borg-backup[14433]: ==== END backup, итоговый код: 0 ====
root@client:~#

root@client:~# cat /etc/logrotate.d/borg-backup
# Managed by Ansible
/var/log/borg-backup.log {
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
root@client:~#

root@client:~# logrotate -d /etc/logrotate.d/borg-backup
WARNING: logrotate in debug mode does nothing except printing debug messages!  Consider using verbose mode (-v) instead if this is not what you want.

reading config file /etc/logrotate.d/borg-backup
Reading state from file: /var/lib/logrotate/status
state file /var/lib/logrotate/status does not exist
Allocating hash table for state file, size 64 entries

Handling 1 logs

rotating pattern: /var/log/borg-backup.log  after 1 days (14 rotations)
empty log files are not rotated, old logs are removed
considering log /var/log/borg-backup.log
error: skipping "/var/log/borg-backup.log" because parent directory has insecure permissions (It's world writable or writable by group which is not "root") Set "su" directive in config file to tell logrotate which user/group should be used for rotation.
Creating new state
root@client:~#
```

Что искать в логах: `==== START backup`, блок `Archive name: etc-...`, `terminating with success status, rc 0` для `create`, `prune` и `check`, строка `==== END backup, итоговый код: 0 ====`.

### 9.6. Проверка политики prune (без удаления)

```bash
borg prune --dry-run --list --prefix etc- \
    --keep-daily 90 --keep-monthly 9 "$BORG_REPO"
```

```Bash
root@client:~# borg prune --dry-run --list --prefix etc- \
    --keep-daily 90 --keep-monthly 9 "$BORG_REPO"
Keeping archive (rule: daily #1):        etc-2026-10-15_08:34:03              Thu, 2026-10-15 08:34:04 [58ab485f925cee8b941b2db48fbec0d0c267486f90073842f8c2a91f2690750f]
Would prune:                             etc-2026-10-15_08:29:02              Thu, 2026-10-15 08:29:02 [abca14a1805b9cf708adf5804b01b43b4559b563e895bf22bf99cc8016b26964]
Would prune:                             etc-2026-10-15_08:24:01              Thu, 2026-10-15 08:24:02 [8b30c98f0cb2d9ea4c3d72d3cf57bb2027cc5239d77079cd04fa7c15b71ae7e3]
Would prune:                             etc-2026-10-15_08:19:00              Thu, 2026-10-15 08:19:00 [c84b3d8e6b2718d6249b71f62f7d0368b3d77c94736e140c8156cef1a99aad27]
Would prune:                             etc-2026-10-15_08:13:59              Thu, 2026-10-15 08:14:00 [6d06a89afc1a28a38daf26b4736d6e508b60f01f41dd32202d601a92e8671002]
Would prune:                             etc-2026-10-15_08:08:58              Thu, 2026-10-15 08:08:58 [15a301e7b3e1d479d5126219c89be72658b313bfe3041e62b9a8a93d1736eb80]
Would prune:                             etc-2026-10-15_08:03:57              Thu, 2026-10-15 08:03:58 [596d4fe269904bf565aa2d4e7a0dd5e266ebb38c560397128a979ef089a45f0b]
Would prune:                             etc-2026-10-15_07:58:56              Thu, 2026-10-15 07:58:56 [7e7f7ec3ee868aaf26e2b245b6685c634e7a59d7c997efbc07643d59dda2be09]
Would prune:                             etc-2026-10-15_07:53:55              Thu, 2026-10-15 07:53:56 [da0d8eda85ee8390745a69c0d1d09b1a2101ed8fa5e5dc132ada74488db12d2e]
Would prune:                             etc-2026-10-15_07:48:54              Thu, 2026-10-15 07:48:54 [3679cfea0862b8c1864d4e330786593ff96b6a83dea866f1d83f119e2566a3dc]
Would prune:                             etc-2026-10-15_07:43:53              Thu, 2026-10-15 07:43:53 [ba7f5004a34d11d47ba38d876c8464b1360b7878dd27716981688dc8b73ab144]
Would prune:                             etc-2026-10-15_07:38:52              Thu, 2026-10-15 07:38:53 [5847a630e8c8ba6a8d48ad1096de2eaaade816e45c572c89aa72f35d64c57a1b]
Keeping archive (rule: daily[oldest] #2): etc-2026-10-15_06:38:40              Thu, 2026-10-15 06:38:41 [576fb3f215178f7c14b6ec7a32ab8043f215ed91999b778e665496f5af3a928e]
root@client:~#
```

* * *

## 10\. Восстановление `/etc` из бэкапа

### 10.1. Подготовка: маркерный файл

Создаём файл, которого заведомо нет в старых копиях, и ждём следующего запуска бэкапа (до 5 минут), чтобы он попал в архив:

```bash
echo "restore-test $(date)" > /etc/restore-test.txt
systemctl start borg-backup.service      # или дождаться таймера
```

### 10.2. Остановка бэкапа

```bash
systemctl stop borg-backup.timer
systemctl stop borg-backup.service       # если выполняется прямо сейчас
systemctl list-timers --all | grep borg  # убедиться, что таймера нет
```

### 10.3. Выбор архива

```bash
borg list "$BORG_REPO"
ARCHIVE=$(borg list --short --last 1 "$BORG_REPO")   # последний архив
echo "$ARCHIVE"
borg list "$BORG_REPO::$ARCHIVE" | grep restore-test
```

### 10.4. «Авария»: перемещаем `/etc`

Без `/etc` не будут работать `ssh` (нужны `passwd`, `group`, `nsswitch.conf`, `ssh_config`) и, как следствие, обращение borg к серверу. Поэтому перед восстановлением оставляем минимальный набор файлов:

```bash
mv /etc /etc.broken
mkdir /etc
cp -a /etc.broken/{passwd,group,nsswitch.conf,hosts} /etc/
cp -a /etc.broken/ssh /etc/
ls /etc            # убеждаемся, что остался только минимум
```

> Если `ssh` на следующем шаге ругается на отсутствие какого-то файла — добавьте его в `/etc` из `/etc.broken`

### 10.5. Восстановление

Архив хранит пути без ведущего `/` (`etc/...`), поэтому извлекаем из корня:

```bash
cd /
borg extract --list "$BORG_REPO::$ARCHIVE" etc
```

### 10.6. Проверка результата

```bash
ls /etc | wc -l
cat /etc/restore-test.txt                 # маркерный файл вернулся
diff -rq /etc /etc.broken | head -n 30    # отличия только в файлах, изменённых после снятия копии
```

```
root@client:/# ls /etc | wc -l
181
root@client:/# cat /etc/restore-test.txt
restore-test Thu Oct 15 08:38:05 UTC 2026
```

### 10.7. Возврат к штатной работе

```bash
systemctl daemon-reload
systemctl start borg-backup.timer
systemctl list-timers --all | grep borg
rm -rf /etc.broken     # после убеждения, что всё в порядке
```

### Альтернатива: точечное восстановление и просмотр архива

```bash
# достать один файл
borg extract "$BORG_REPO::$ARCHIVE" etc/hostname

# смонтировать архив и посмотреть как каталог (нужен fuse)
mkdir -p /mnt/borg
borg mount "$BORG_REPO::$ARCHIVE" /mnt/borg
ls /mnt/borg/etc
borg umount /mnt/borg
```

* * *

## 11\. Заметки

- **Ключ репозитория.** При `repokey` ключ лежит в самом репозитории. Если потерять и репозиторий, и пароль — данные не восстановить. Рекомендуется сделать экспорт ключа и хранить отдельно:
    
    ```bash
    borg key export "$BORG_REPO" /root/borg-repo-key.txt
    ```
    
- **Пароль.** В демо пароль в `group_vars/all.yml` открытым текстом; в реальной эксплуатации — `ansible-vault encrypt_string` или внешнее хранилище секретов.
    
- **`borg check` каждые 5 минут.** Полная проверка репозитория по таймеру каждые 5 минут слишком тяжёлая, поэтому в скрипте проверяется только последний архив. Полную проверку (`borg check "$BORG_REPO"`) разумно выносить в отдельный таймер раз в сутки/неделю.
    
- **Блокировка репозитория.** Borg сам ставит lock на репозиторий, а `Type=oneshot` + `OnUnitActiveSec` не запускают сервис параллельно с самим собой.
    
- **Ограничение доступа.** Для ещё большей защиты от «эффекта вируса-шифровальщика» можно добавить в `authorized_keys` ключ `--append-only` (тогда `prune` придётся выполнять отдельно, с админским ключом).
    
- **Особенность политики хранения.** Из-за `--keep-daily` в течение дня остаётся одна, последняя копия; правило `--keep-within` в скрипте — временная мера для демонстрации 5-минутных снимков.
    

## 12\. Полезные источники

- https://habr.com/ru/company/flant/blog/420055/
- https://www.opennet.ru/tips/3180_borg_backup.shtml
- https://borgbackup.readthedocs.io/en/stable/usage/general.html
- https://borgbackup.readthedocs.io/en/stable/usage/prune.html
- https://github.com/alisianoi/borg-systemd
- https://dextervolkman.com/posts/borg_backups