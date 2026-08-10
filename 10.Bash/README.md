## Требования к скрипту

1.  **Защита от параллельного запуска** (`flock`): предотвращает одновременное выполнение нескольких копий.
    
2.  **Отчет только по новым данным**: скрипт запоминает номер последней прочитанной строки в файле состояния (`/tmp/last_line.txt`).
    
3.  **Содержание отчета**:
    
    - Обрабатываемый временной диапазон.
        
    - TOP-10 IP-адресов по количеству запросов.
        
    - TOP-10 запрашиваемых URL.
        
    - HTTP-коды ответов и их количество.
        
    - Ошибки веб-сервера и приложения (коды `4xx` и `5xx`).
        
4.  **Гарантия сохранения состояния**: номер обработанной строки обновляется только при успешной отправке письма (`&& echo "$TOTAL_LINES" > "$STATE"`).
    

## 1\. Установка и настройка SMTP-клиента (msmtp)

Для отправки писем через внешний SMTP-сервер (например, Яндекс) используются утилиты `msmtp` и `mailutils`.

### 1.1. Установка необходимых пакетов

```
apt update && apt install -y msmtp msmtp-mta mailutils
```

### 1.2. Создание файла конфигурации `/etc/msmtprc`

Создайте или отредактируйте файл `/etc/msmtprc`:

```
nano /etc/msmtprc
```

Вставьте следующую конфигурацию:

Ini, TOML

```
# Общие настройки
defaults
auth           on
tls            on
tls_starttls   off
tls_trust_file /etc/ssl/certs/ca-certificates.crt
syslog         LOG_MAIL

# Учетная запись по умолчанию
account        default
host           smtp.yandex.ru
port           465
from           Sevastyanov.KV@yandex.ru
user           Sevastyanov.KV@yandex.ru
password       ВАШ_ПАРОЛЬ_ПРИЛОЖЕНИЯ
allow_from_override off
```

> **Примечание:**
> 
> - Поле `password` должно содержать **Пароль приложения** (генерируется в *Яндекс ID -> Безопасность -> Пароли приложений -> Почта SMTP*).
>     
> - Параметр `allow_from_override off` запрещает утилите `mail` подменять заголовок `From`, предотвращая блокировку со стороны Яндекса.
>     
> - Параметр `syslog LOG_MAIL` направляет логи отправки в системный журнал без ошибок доступа.
>     

### 1.3. Настройка прав доступа

```
chmod 600 /etc/msmtprc
chown root:root /etc/msmtprc
```

### 1.4. Проверка отправки тестового письма

```
echo "Тестовое сообщение" | mail -s "Проверка SMTP" Sevastyanov.KV@yandex.ru
```

## 2\. Создание Bash-скрипта отчета

Создайте файл скрипта `/opt/parce_script.sh`:

```
nano /opt/parce_script.sh
```

Вставьте код скрипта:

```
#!/bin/bash

# ==============================================================================
# Скрипт отчета о работе веб-сервера
# ==============================================================================

# 1. Защита от повторного запуска (flock)
exec 9>/tmp/web_report.lock; flock -n 9 || exit 1

# Настройки
LOG="/home/kirills/access_.log"         # Путь к лог-файлу веб-сервера
EMAIL="sevastyanov.kv@yandex.ru"        # Email получателя отчета
STATE="/tmp/last_line.txt"              # Файл хранения номера последней строки

# 2. Определение и вырезка новых строк
LAST_LINE=$(cat "$STATE" 2>/dev/null || echo 0)
TOTAL_LINES=$(wc -l < "$LOG")

# Если новых записей нет — завершаем работу
[ "$TOTAL_LINES" -le "$LAST_LINE" ] && exit 0

# Извлекаем только новые строки с момента последнего запуска
NEW_LOGS=$(sed -n "$((LAST_LINE + 1)),${TOTAL_LINES}p" "$LOG")

# 3. Формирование отчета и отправка по почте
(
  echo "Отчет о работе веб-сервера"
  echo "Период отчета: $(date -d '1 hour ago' '+%Y-%m-%d %H:00') — $(date '+%Y-%m-%d %H:00')"
  echo "=================================================="

  echo -e "\n--- TOP 10 IP-АДРЕСОВ ---"
  echo "$NEW_LOGS" | awk '{print $1}' | sort | uniq -c | sort -nr | head -n 10

  echo -e "\n--- TOP 10 ЗАПРАШИВАЕМЫХ URL ---"
  echo "$NEW_LOGS" | awk '{print $7}' | sort | uniq -c | sort -nr | head -n 10

  echo -e "\n--- HTTP КОДЫ ОТВЕТОВ ---"
  echo "$NEW_LOGS" | awk '{print $9}' | sort | uniq -c | sort -nr

  echo -e "\n--- ОШИБКИ ВЕБ-СЕРВЕРА И ПРИЛОЖЕНИЯ (4xx и 5xx) ---"
  echo "$NEW_LOGS" | awk '$9 ~ /^[45]/ {print $0}' | head -n 10

) | mail -s "Веб-отчет за час" "$EMAIL" && echo "$TOTAL_LINES" > "$STATE"
```

Сделайте файл исполняемым:

```
chmod +x /opt/parce_script.sh
```

## 3\. Настройка планировщика CRON

Откройте планировщик задач:

```
crontab -e
```

Добавьте строчку для выполнения скрипта в начало каждого часа:

Фрагмент кода

```
0 * * * * /opt/parce_script.sh >/dev/null 2>&1
```

## 4\. Тестирование и Диагностика

### Ручной запуск и сброс состояния

Чтобы протестировать скрипт повторно на том же логе, сбросьте файл состояния и запустите скрипт вручную:

```
rm -f /tmp/last_line.txt
/opt/parce_script.sh
```

### Просмотр почты

&nbsp;

```
Период отчета: 2026-08-08 07:00 — 2026-08-08 08:00
--------------------------------------------------

--- TOP IP ---
     45 93.158.167.130
     39 109.236.252.130
     37 212.57.117.19
     33 188.43.241.106
     31 87.250.233.68
     24 62.75.198.172
     22 148.251.223.21
     20 185.6.8.9
     17 217.118.66.161
     16 95.165.18.146

--- TOP URL ---
    157 /
    120 /wp-login.php
     57 /xmlrpc.php
     26 /robots.txt
     12 /favicon.ico
     11 400
      9 /wp-includes/js/wp-embed.min.js?ver=5.0.4
      7 /wp-admin/admin-post.php?page=301bulkoptions
      7 /1
      6 /wp-content/uploads/2016/10/robo5.jpg

--- HTTP КОДЫ ---
    498 200
     95 301
     51 404
     11 "-"
      7 400
      3 500
      2 499
      1 405
      1 403
      1 304

--- ОШИБКИ (4xx и 5xx) ---
93.158.167.130 - - [14/Aug/2019:05:02:20 +0300] "GET / HTTP/1.1" 404 169 "-" "Mozilla/5.0 (compatible; YandexMetrika/2.0; +http://yandex.com/bots yabs01)"rt=0.000 uct="-" uht="-" urt="-"
87.250.233.68 - - [14/Aug/2019:05:04:20 +0300] "GET / HTTP/1.1" 404 169 "-" "Mozilla/5.0 (compatible; YandexMetrika/2.0; +http://yandex.com/bots yabs01)"rt=0.000 uct="-" uht="-" urt="-"
107.179.102.58 - - [14/Aug/2019:05:22:10 +0300] "GET /wp-content/plugins/uploadify/readme.txt HTTP/1.1" 404 200 "http://dbadmins.ru/wp-content/plugins/uploadify/readme.txt" "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/42.0.2311.152 Safari/537.36"rt=0.000 uct="-" uht="-" urt="-"
193.106.30.99 - - [14/Aug/2019:06:02:50 +0300] "GET /wp-includes/ID3/comay.php HTTP/1.1" 500 595 "-" "Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36"rt=0.000 uct="-" uht="-" urt="-"
87.250.244.2 - - [14/Aug/2019:06:07:07 +0300] "GET / HTTP/1.1" 404 169 "-" "Mozilla/5.0 (compatible; YandexMetrika/2.0; +http://yandex.com/bots yabs01)"rt=0.000 uct="-" uht="-" urt="-"
77.247.110.165 - - [14/Aug/2019:06:13:53 +0300] "HEAD /robots.txt HTTP/1.0" 404 0 "-" "-"rt=0.018 uct="-" uht="-" urt="-"
87.250.233.76 - - [14/Aug/2019:06:45:20 +0300] "GET / HTTP/1.1" 404 169 "-" "Mozilla/5.0 (compatible; YandexMetrika/2.0; +http://yandex.com/bots yabs01)"rt=0.000 uct="-" uht="-" urt="-"
71.6.199.23 - - [14/Aug/2019:07:07:19 +0300] "GET /robots.txt HTTP/1.1" 404 3652 "-" "-"rt=0.000 uct="-" uht="-" urt="-"
71.6.199.23 - - [14/Aug/2019:07:07:20 +0300] "GET /sitemap.xml HTTP/1.1" 404 3652 "-" "-"rt=0.000 uct="-" uht="-" urt="-"
71.6.199.23 - - [14/Aug/2019:07:07:20 +0300] "GET /.well-known/security.txt HTTP/1.1" 404 3652 "-" "-"rt=0.000 uct="-" uht="-" urt="-"
```