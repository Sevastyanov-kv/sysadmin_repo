# Домашнее задание: Docker

## Цель

Освоить базовые принципы работы с Docker, научиться создавать, настраивать и управлять контейнерами.

* * *

## 1\. Установка Docker

Установка выполнена по официальной инструкции для Ubuntu:  
https://docs.docker.com/engine/install/ubuntu/

Кратко порядок действий:

```bash
# Удаление старых версий (если были)
sudo apt-get remove docker docker-engine docker.io containerd runc

# Установка зависимостей
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# Добавление официального GPG-ключа Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Добавление репозитория
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Установка Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверка установки:

```bash
sudo docker run hello-world
```

* * *

## 2\. Установка Docker Compose

Docker Compose устанавливался как плагин (`docker-compose-plugin`), что видно из команды установки выше — он ставится в одном пакете с Docker Engine.

Проверка версии:

```bash
docker compose version

kirills@otus:~$ docker compose version
Docker Compose version v5.5.1

```

* * *

## 3\. Кастомный образ nginx на базе Alpine

### Структура проекта

```
mkdir -p ~/docker-nginx-hw
cd ~/docker-nginx-hw

docker-nginx-hw/
├── Dockerfile
└── index.html
```

### Dockerfile

```dockerfile
FROM nginx:alpine

# Удаляем дефолтную страницу и копируем свою
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### index.html (кастомная страница)

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Custom Nginx Page</title>
</head>
<body>
    <h1>🚀 Custom Nginx on Alpine</h1>
    <p>Домашнее задание по курсу «Администратор Linux. Professional»</p>
    <p>Тема: Docker</p>
</body>
</html>
```

### Сборка и запуск образа

```bash
# Сборка образа
docker build -t  adminrepo/nginx-custom:v1 .

# Запуск контейнера с пробросом порта
docker run -d -p 8080:80 --name nginx-custom  adminrepo/nginx-custom:v1

# Проверка списка запущенных контейнеров
docker ps

# Проверка отдаваемой страницы
curl http://localhost:8080
```

После выполнения этих команд по адресу `http://192.168.1.57:8080/` открывается кастомная страница вместо стандартной страницы nginx.

Дополнительные полезные команды при отладке:

```bash
docker logs nginx-custom       # логи контейнера
docker inspect nginx-custom    # подробная информация о контейнере
docker exec -it nginx-custom sh  # вход в оболочку контейнера (alpine — sh, не bash)
```

* * *

## 4\. Разница между контейнером и образом (вывод)

**Образ (image)** — это неизменяемый шаблон, статичный «слепок» файловой системы и метаданных (какие команды выполнить при старте, какие порты открыть и т.д.). Образ состоит из слоёв (layers), каждый из которых соответствует инструкции в Dockerfile (`FROM`, `COPY`, `RUN` и т.д.). Образ сам по себе ничего не выполняет — это шаблон для создания контейнеров, хранится в реестре (Docker Hub, локальный кэш) и может быть использован многократно.

**Контейнер (container)** — это работающий (или остановленный) экземпляр образа. При запуске образа Docker добавляет поверх его слоёв ещё один — тонкий, доступный для записи (writable layer), — и запускает в изолированном окружении (с помощью namespaces и cgroups) процесс, указанный в `CMD`/`ENTRYPOINT`. Все изменения, которые происходят во время работы контейнера (новые файлы, изменения в файлах), пишутся именно в этот верхний слой и теряются при удалении контейнера, если они не вынесены в volume.

**Кратко:**

|     | Образ | Контейнер |
| --- | --- | --- |
| Природа | Статичный шаблон | Запущенный процесс на основе образа |
| Изменяемость | Неизменяемый (immutable) | Имеет изменяемый верхний слой |
| Хранение | Реестр / локальный кэш | Существует, пока не удалён |
| Аналогия | Класс (в ООП) / установочный дистрибутив | Объект класса / запущенная программа |

Из одного образа можно запустить сколько угодно независимых контейнеров.

* * *

## 5\. Можно ли в контейнере собрать ядро?

Да, ядро можно собрать в контейнере.

Контейнер использует ядро хостовой операционной системы, но для компиляции ядра ему не требуется запускать собственное ядро. В контейнер можно установить необходимые инструменты сборки (компилятор GCC, make, библиотеки и исходный код ядра) и выполнить сборку.

Однако есть нюансы:

- Контейнер не предоставляет полноценную виртуальную машину и не может загрузить собранное ядро самостоятельно.
    
- Для сборки ядра нужны соответствующие права доступа и зависимости.
    
- Если требуется протестировать или запустить собранное ядро, обычно используют виртуальную машину или перезагрузку хостовой системы.
    

Итог: собрать ядро в контейнере можно, а вот загрузить его как ядро контейнера — нельзя, поскольку контейнеры используют ядро хоста.

* * *

## 6\. Публикация образа в Docker Hub

```bash
# Авторизация в Docker Hub
docker login

# Отправка образа в Docker Hub
docker push <docker_hub_login>/nginx-custom:v1
```

**Ссылка на репозиторий в Docker Hub:**  
`https://hub.docker.com/repository/docker/adminrepo/nginx-custom/general`