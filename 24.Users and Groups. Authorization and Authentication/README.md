# ДЗ: Vagrant-стенд с PAM

## 1\. Цель

Научиться создавать пользователей и добавлять им ограничения.

## 2\. Текст задания

Ограничить доступ к системе для всех пользователей, кроме группы администраторов, в выходные дни (суббота и воскресенье), за исключением праздничных дней.

⭐️ **Задание повышенной сложности** (необязательное): предоставить определённому пользователю доступ к Docker и право перезапускать Docker-сервис.

## 3\. Структура репозитория

```text
.
├── README.md
└── Vagrantfile
```

## 4\. Теория (кратко)

Почти все операционные системы Linux — многопользовательские. Администратор Linux должен уметь создать и настраивать пользователей.

В Linux есть 3 группы пользователей:

- Администраторы — привилегированные пользователи с полным доступом к системе. По умолчанию в ОС есть такой пользователь — root
    
- Локальные пользователи — их учетные записи создает администратор, их права ограничены. Администраторы могут изменять права локальных пользователей
    
- Системные пользователи — учетный записи, которые создаются системой для внутренних процессов и служб. Например пользователь — nginx
    

У каждого пользователя есть свой уникальный идентификатор — UID.

Чтобы упростить процесс настройки прав для новых пользователей, их объединяют в группы. Каждая группа имеет свой набор прав и ограничений. Любой пользователь, создаваемый или добавляемый в такую группу, автоматически их наследует. Если при добавлении пользователя для него не указать группу, то у него будет своя, индивидуальная группа — с именем пользователя. Один пользователь может одновременно входить в несколько групп.

Информацию о каждом пользователе сервера можно посмотреть в файле /etc/passwd

Для более точных настроек пользователей можно использовать подключаемые модули аутентификации (PAM)

PAM (Pluggable Authentication Modules - подключаемые модули аутентификации) — набор библиотек, которые позволяют интегрировать различные методы аутентификации в виде единого API.

PAM решает следующие задачи:

- Аутентификация — процесс подтверждения пользователем своей подлинности. Например: ввод логина и пароля, ssh-ключ и т д.
    
- Авторизация — процесс наделения пользователя правами
    
- Отчетность — запись информации о произошедших событиях
    

PAM может быть реализован несколькими способами:

- Модуль pam_time — настройка доступа для пользователя с учетом времени
    
- Модуль pam_exec — настройка доступа для пользователей с помощью скриптов
    

&nbsp;

## 5\. Vagrantfile

```ruby
MACHINES = {
  :"pam" => {
              :box_name => "ubuntu/jammy64",
              :cpus => 2,
              :memory => 1024,
              :ip => "192.168.57.10",
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.network "private_network", ip: boxconfig[:ip]

    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end

      box.vm.provision "shell", inline: <<-SHELL
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
          systemctl restart sshd.service
      SHELL
    end
  end
end
```

> **Отличие от методички:** добавлена вторая строка `sed`. В cloud-образе `ubuntu/jammy64` файл `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` содержит `PasswordAuthentication no` и перебивает основной `sshd_config` (а в самом `sshd_config` нужная строка закомментирована, поэтому первый `sed` ничего не меняет). Без этой правки вход по паролю не работает: `Permission denied (publickey)`.

Пояснения:

- в ВМ добавлен дополнительный сетевой интерфейс (`private_network`, IP `192.168.57.10`), чтобы подключаться к ней по SSH с хоста;
- для удобства в настройках SSH разрешена аутентификация по паролю (`PasswordAuthentication yes`), затем перезапускается служба SSH;
- проверить итоговое значение, которое использует sshd, можно командой `sshd -T | grep -i passwordauthentication`.

```bash
vagrant up
```

* * *

## 6\. Запрет входа в выходные для всех, кроме группы admin

### 6.1 Подключаемся к ВМ и становимся root

```bash
vagrant ssh
sudo -i
```

### 6.2 Создаём пользователей `otusadm` и `otus`

```bash
useradd -m -s /bin/bash otusadm && useradd -m -s /bin/bash otus
grep otus /etc/passwd

-m (--create-home) создаёт домашний каталог пользователя, то есть /home/otusadm и /home/otus
-s /bin/bash (--shell) задаёт оболочку, которая запускается при входе
```

### 6.3 Задаём пользователям пароли

```bash
echo "otusadm:pass12" | chpasswd && echo "otus:pass12" | chpasswd
```

> Для примера у `otus` и `otusadm` одинаковые пароли.

### 6.4 Создаём группу `admin`

```bash
groupadd -f admin
-f (--force) означает «не считать ошибкой, если группа уже существует». Без него повторный запуск groupadd admin на существующей группе завершится ошибкой

grep admin /etc/group
getent group admin
```

### 6.5 Добавляем `vagrant`, `root` и `otusadm` в группу `admin`

```bash
usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin

usermod	изменить существующего пользователя
otusadm	имя пользователя, которого меняем
-G admin (--groups) список дополнительных групп пользователя, здесь одна группа admin
-a	 (--append) добавить к уже имеющимся группам, а не заменять их
```

> Пользователь `otusadm` просто добавлен в группу `admin`. Это **не делает** его администратором.

### 6.6 Проверяем подключение по SSH

С хостовой машины:

```bash
ssh otus@192.168.57.10
ssh otusadm@192.168.57.10
```

Вводим пароль `pass12`. Если всё сделано правильно, подключиться можно под обоими пользователями.

### 6.7 Проверяем состав группы `admin`

```bash
cat /etc/group | grep admin
```

Пример вывода (номер GID у вас может отличаться):

```text
root@pam:~# cat /etc/group | grep admin
admin:x:117:otusadm,root,vagrant
root@pam:~#
```

> Информация о группах и их участниках хранится в `/etc/group`, пользователи перечислены через запятую.

### 6.8 Создаём скрипт `/usr/local/bin/login.sh`

```bash
nano /usr/local/bin/login.sh
```

```bash
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
```

Логика скрипта: если сегодня суббота или воскресенье, проверяем, входит ли пользователь в группу `admin`; если не входит — подключение запрещено. Во всех остальных случаях подключение разрешено.

### 6.9 Даём права на исполнение

```bash
chmod +x /usr/local/bin/login.sh

root@pam:~# ls -la /usr/local/bin/login.sh
-rwxr-xr-x 1 root root 719 Oct  1 09:03 /usr/local/bin/login.sh
```

### 6.10 Подключаем `pam_exec` и скрипт в `/etc/pam.d/sshd`

```bash
nano /etc/pam.d/sshd
```

Добавляем строку (рядом с остальными `auth`\-правилами, после `@include common-auth`):

```text
auth required pam_exec.so debug /usr/local/bin/login.sh
```

Проверяем:

```bash
grep pam_exec /etc/pam.d/sshd

root@pam:~# grep pam_exec /etc/pam.d/sshd
auth required pam_exec.so debug /usr/local/bin/login.sh
root@pam:~#
```

### 6.11 Проверка работы

Если работа выполняется в выходные — можно сразу пробовать подключиться. Если нет — меняем время в ОС на субботу, например 03 Октября 2026 года:

```bash
sudo date 100312302026.00
date
```

> Команда `sudo date 082712302022.00` расшифровывается следующим образом:
> 
> - **`08`** (`MM`) — месяц: август (**08**)
>     
> - **`27`** (`DD`) — день месяца: **27**\-е число
>     
> - **`12`** (`hh`) — часы: **12** часов
>     
> - **`30`** (`mm`) — минуты: **30** минут
>     
> - **`2022`** (`CCYY`) — год: **2022** (**CC** — век 20, **YY** — год 22)
>     
> - **`.00`** (`.ss`) — секунды: **00** секунд (указываются после точки)
>     
> 
> Для того, чтобы применилась дата, нужно остановить службу: sudo systemctl status vboxadd-service

&nbsp;

Пробуем подключиться с хоста:

```bash
ssh otus@192.168.57.10
```

```text
PS C:\Users\HomeStation> ssh otus@192.168.57.10
otus@192.168.57.10's password:
Permission denied, please try again.
otus@192.168.57.10's password:
```

Пользователь `otus` (не входит в `admin`) получает отказ.

```bash
ssh otusadm@192.168.57.10
```

```text
otusadm@192.168.57.10's password:
Welcome to Ubuntu 22.04 ...
otusadm@pam:~$
```

Пользователь `otusadm` (входит в `admin`) подключается без проблем.

* * *

## 7\. Полезные ссылки

- [Управление пользователями в Linux](https://firstvds.ru/technology/linux-user-management)
- `man pam.d`, `man pam_exec`, `man sudoers`