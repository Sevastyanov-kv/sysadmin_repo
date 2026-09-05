# Домашнее задание: Автоматизация администрирования. Ansible

## Цель

Первые шаги с Ansible: с помощью Ansible-роли установить и настроить nginx на удалённом сервере.

## Примечание про стенд

Изначально планировалось поднять стенд через Vagrant (box `generic/ubuntu2204`), но скачивание бокса  
из Vagrant Cloud не работает без VPN (`Recv failure: Connection was reset` при обращении к  
`vagrantcloud.com`). Поэтому вместо Vagrant использованы уже готовые виртуальные машины в одной  
локальной сети:

| Роль | Хост | IP  | Пользователь |
| --- | --- | --- | --- |
| Управляющий (control host) | otus | 192.168.1.57 | kirills |
| Целевой (managed host) | otus1 | 192.168.1.38 | kirills |

Ansible ставится **только на управляющую машину** (`otus`), на целевой машине (`otus1`) никакого  
Ansible нет — она принимает команды только по SSH.

* * *

## 1\. Установка Ansible на управляющей машине (otus, 192.168.1.57)

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

## 2\. Настройка SSH-доступа с otus на otus1

```bash
# на otus (управляющей машине)
ssh-keygen -t ed25519 -C "ansible-control"   # если ключа ещё нет
ssh-copy-id kirills@192.168.1.38
```

Проверка входа без пароля:

```bash
ssh kirills@192.168.1.38
```

## 3\. Структура проекта

mkdir -p ~/ansible

```
ansible/
├── ansible.cfg
├── nginx.yml
├── staging/
│   └── hosts
└── roles/
    └── nginx/
        ├── defaults/
        │   └── main.yml
        ├── vars/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        └── templates/
            └── nginx.conf.j2
```

### 3.1. `ansible.cfg`

```ini
[defaults]
inventory = staging/hosts
host_key_checking = False
retry_files_enabled = False
roles_path = roles
```

### 3.2. `staging/hosts`

```ini
[web]
otus1 ansible_host=192.168.1.38 ansible_user=kirills
```

Проверка доступности хоста:

```bash
ansible web -m ping
```

результат:

```
kirills@otus:~/ansible$ ansible web -m ping
[WARNING]: Host 'otus1' is using the discovered Python interpreter at '/usr/bin/python3.14', but future installation of another Python interpreter could cause a different interpreter to be discovered. See https://docs.ansible.com/ansible-core/2.20/reference_appendices/interpreter_discovery.html for more information.
otus1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}

```

> Если под пользователем `kirills` на `otus1` sudo спрашивает пароль, а не passwordless — запускать  
> playbook с флагом `--ask-become-pass` и ввести пароль в момент выполнения, либо один раз прописать  
> `ansible_become_pass` в inventory (менее безопасно, пароль хранится в открытом виде).

* * *

## 4\. Ansible-роль `nginx`

### 4.1. `roles/nginx/defaults/main.yml`

Значение по умолчанию для порта (можно переопределить на уровне playbook / inventory / `-e`):

```yaml
---
nginx_listen_port: 8080
```

### 4.2. `roles/nginx/vars/main.yml`

```yaml
---
nginx_package: "nginx"
nginx_config_path: "/etc/nginx/nginx.conf"
```

### 4.3. `roles/nginx/tasks/main.yml`

```yaml
---
- name: NGINX | Install NGINX (apt)
  apt:
    name: "{{ nginx_package }}"
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"
  tags:
    - nginx-install

- name: NGINX | Install NGINX (yum)
  yum:
    name: "{{ nginx_package }}"
    state: present
  when: ansible_os_family == "RedHat"
  tags:
    - nginx-install

- name: NGINX | Create NGINX config file from template
  template:
    src: templates/nginx.conf.j2
    dest: "{{ nginx_config_path }}"
    owner: root
    group: root
    mode: "0644"
  notify: restart nginx
  tags:
    - nginx-configuration

- name: NGINX | Enable NGINX service in systemd
  systemd:
    name: nginx
    enabled: yes
  tags:
    - nginx-service
```

### 4.4. `roles/nginx/handlers/main.yml`

```yaml
---
- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes

- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```

### 4.5. `roles/nginx/templates/nginx.conf.j2`

```nginx
events {
    worker_connections 1024;
}

http {
    server {
        listen {{ nginx_listen_port }} default_server;
        server_name default_server;
        root /usr/share/nginx/html;

        location / {
        }
    }
}
```

* * *

## 5\. Playbook `nginx.yml`

```yaml
---
- name: NGINX | Install and configure NGINX
  hosts: web
  become: true
  vars:
    nginx_listen_port: 8080
  roles:
    - nginx
```

* * *

## 6\. Запуск

```bash
cd ansible
ansible-playbook nginx.yml
```

Пример вывода:

```ansible
kirills@otus:~/ansible$ ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] ***************************************************************************

TASK [Gathering Facts] ***********************************************************************************************
[WARNING]: Host 'otus1' is using the discovered Python interpreter at '/usr/bin/python3.14', but future installation of another Python interpreter could cause a different interpreter to be discovered. See https://docs.ansible.com/ansible-core/2.20/reference_appendices/interpreter_discovery.html for more information.
ok: [otus1]

TASK [nginx : NGINX | Install NGINX (apt)] ***************************************************************************
[WARNING]: Deprecation warnings can be disabled by setting `deprecation_warnings=False` in ansible.cfg.
[DEPRECATION WARNING]: INJECT_FACTS_AS_VARS default to `True` is deprecated, top-level facts will not be auto injected after the change. This feature will be removed from ansible-core version 2.24.
Origin: /home/kirills/ansible/roles/nginx/tasks/main.yml:7:9

5     state: present
6     update_cache: yes
7   when: ansible_os_family == "Debian"
          ^ column 9

Use `ansible_facts["fact_name"]` (no `ansible_` prefix) instead.

changed: [otus1]

TASK [nginx : NGINX | Install NGINX (yum)] ***************************************************************************
[DEPRECATION WARNING]: INJECT_FACTS_AS_VARS default to `True` is deprecated, top-level facts will not be auto injected after the change. This feature will be removed from ansible-core version 2.24.
Origin: /home/kirills/ansible/roles/nginx/tasks/main.yml:15:9

13     name: "{{ nginx_package }}"
14     state: present
15   when: ansible_os_family == "RedHat"
           ^ column 9

Use `ansible_facts["fact_name"]` (no `ansible_` prefix) instead.

skipping: [otus1]

TASK [nginx : NGINX | Create NGINX config file from template] ********************************************************
changed: [otus1]

TASK [nginx : NGINX | Enable NGINX service in systemd] ***************************************************************
ok: [otus1]

RUNNING HANDLER [nginx : restart nginx] ******************************************************************************
changed: [otus1]

PLAY RECAP ***********************************************************************************************************
otus1                      : ok=5    changed=3    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

kirills@otus:~/ansible$

```

## 7\. Проверка результата

```bash
curl http://192.168.1.38:8080


kirills@otus:~/ansible$ curl http://192.168.1.38:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
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
kirills@otus:~/ansible$

```

или в браузере: `http://192.168.1.38:8080`.

Дополнительные проверки:

```bash
ansible web -m shell -a "systemctl is-enabled nginx"
ansible web -m shell -a "ss -tulnp | grep 8080"
```

* * *

## 8\. Итоги:

- Настройка SSH-доступа между двумя существующими виртуальными машинами, включая  
    копирование публичного ключа через `ssh-copy-id`.
- Настройка `ansible.cfg` и inventory-файла для работы без явного указания параметров подключения  
    в каждой команде.
- Использование ad-hoc команд (`ansible -m ping`, `ansible -m shell`) для проверки состояния хоста.
- Написание Ansible-роли: `defaults`, `vars`, `tasks`, `handlers`, `templates`.
- Использование модуля `template` для генерации конфигурации nginx из Jinja2-шаблона с переменной  
    порта.
- Использование `notify`/`handlers` для перезапуска сервиса только при изменении конфигурации.
- Перевод сервиса в `enabled` через модуль `systemd`.
- Столкнулся с недоступностью Vagrant Cloud без VPN при попытке скачать box `generic/ubuntu2204` —  
    решено использованием уже существующих виртуальных машин вместо Vagrant-стенда.