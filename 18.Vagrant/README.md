# Расширенная настройка дисков и сетей (Vagrant + VirtualBox)

## 1\. Подготовка окружения

Использованы:

- VirtualBox (провайдер ВМ)
- Vagrant (управление ВМ через код)
- Плагин `vagrant-disksize`/встроенный `config.vm.disk` (Vagrant ≥ 2.2) для добавления дисков

Создана директория проекта, в неё помещён `Vagrantfile`.

```bash
# Vagrantfile помещается в эту директорию D:\Обучение\OTUS\18.Vagrant\vagrant_vm
```

* * *

## 2\. Базовая виртуальная машина

Образ: `ubuntu/jammy64` (Ubuntu 22.04 LTS).  
Память ВМ ограничена **1024 МБ** через провайдер VirtualBox:

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.name   = "disks-network-vm"
  vb.memory = 1024
  vb.cpus   = 1
end
```

* * *

## 3\. Добавление дисков

Добавлены два виртуальных диска по 1 ГБ каждый через `config.vm.disk`:

```ruby
config.vm.disk :disk, size: "1GB", name: "disk1"
config.vm.disk :disk, size: "1GB", name: "disk2"
```

Vagrant/VirtualBox подключает их к ВМ как дополнительные блочные устройства (обычно `/dev/sdb` и `/dev/sdc`, поскольку системный диск — `/dev/sda`).

* * *

## 4\. Настройка сети (проброс портов)

Настроен проброс 80 порта гостевой системы на 8080 порт хоста:

```ruby
config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"
```

После `vagrant up` сервис на 80 порту гостевой ВМ становится доступен на хосте по адресу `http://127.0.0.1:8080`.

* * *

## **5\. Провижининг**

Шелл-провижининг выполняет:

1.  Форматирование дисков /dev/sdb и /dev/sdc в файловую систему ext4 (mkfs.ext4 -F).
2.  Создание точек монтирования /mnt/disk1 и /mnt/disk2.
3.  Монтирование дисков в эти директории.
4.  Добавление записей в /etc/fstab для автоматического монтирования при загрузке.
5.  Вывод `df -h` для проверки результата.

* * *

## 6\. Полный текст Vagrantfile

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # Память ВМ — 1024 МБ
  config.vm.provider "virtualbox" do |vb|
    vb.memory = 1024
  end

  # Проброс порта: гостевой 80 -> хостовой 8080
  config.vm.network "forwarded_port", guest: 80, host: 8080

  # Два виртуальных диска по 1 ГБ
  # (sda - системный, sdb - служебный диск VirtualBox 10M,
  #  наши два диска - sdc и sdd)
  config.vm.disk :disk, size: "1GB", name: "disk1"
  config.vm.disk :disk, size: "1GB", name: "disk2"

  # Провижининг: ставим nginx, форматируем диски, монтируем, прописываем в fstab
  config.vm.provision "shell", inline: <<-SHELL
    # Устанавливаем и запускаем nginx (для проверки проброса порта)
    apt-get update -y
    apt-get install -y nginx
    systemctl enable nginx
    systemctl restart nginx

    # Форматируем диски в ext4
    mkfs.ext4 -F /dev/sdc
    mkfs.ext4 -F /dev/sdd

    # Создаём точки монтирования
    mkdir -p /mnt/disk1 /mnt/disk2

    # Монтируем диски
    mount /dev/sdc /mnt/disk1
    mount /dev/sdd /mnt/disk2

    # Добавляем записи в /etc/fstab для автомонтирования
    echo "/dev/sdc  /mnt/disk1  ext4  defaults  0  2" >> /etc/fstab
    echo "/dev/sdd  /mnt/disk2  ext4  defaults  0  2" >> /etc/fstab

    df -h
  SHELL
end
```

* * *

## 7\. Проверка результата

### 7.1. Запуск ВМ

```bash
vagrant up
```

### 7.2. Проверка дисков (`df -h`)

Выполнить внутри ВМ:

```bash
vagrant ssh
df -h
```

В выводе должны присутствовать смонтированные `/mnt/disk1` и `/mnt/disk2` размером ~1 ГБ каждый с типом файловой системы `ext4`.

```
vagrant@ubuntu-jammy:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs            96M  996K   95M   2% /run
/dev/sda1        39G  2.3G   37G   6% /
tmpfs           479M     0  479M   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            96M  4.0K   96M   1% /run/user/1000
vagrant         733G  328G  406G  45% /vagrant
/dev/sdc        974M   24K  907M   1% /mnt/disk1
/dev/sdd        974M   24K  907M   1% /mnt/disk2
```

### 7.3. Проверка проброса порта (`netstat -tulpn | grep 8080`)

Выполнить **на хостовой машине** (не в ВМ):

```bash
PS C:\Users\HomeStation> netstat -ano | findstr 8080
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       28664
PS C:\Users\HomeStation>
```

Должна отображаться строка, что порт 8080 прослушивается (LISTEN) — это VirtualBox пробрасывает трафик на гостевой порт 80, где слушает nginx.

Дополнительно можно проверить сам сервис браузером/curl:

```bash
HomeStation@HOMESTATION MINGW64 /usr/bin
$ curl http://127.0.0.1:8080
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   612  100   612    0     0   323k      0 --:--:-- --:--:-- --:--:--  597k<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

Должна вернуться HTML-страница, запущенная nginx внутри ВМ — это и есть практическая проверка проброса порта.

### 7.4. Проверка автомонтирования через fstab

```bash
vagrant ssh -c "sudo umount /mnt/disk1 /mnt/disk2 && sudo mount -a && df -h"
```

или полный `vagrant reload` — после перезагрузки диски должны монтироваться автоматически благодаря записям в `/etc/fstab`.

&nbsp;