# Уведомления при авторизации на сервере в ваш ТГ бот
Примочка для отправки в ТГ бота данных, кто залогинился на ваш сервер и под каким IP


Настройка логов:
1) Создаем файл nano /etc/logrotate.d/syslog с содержимым
```
   /var/log/syslog {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 root adm
    postrotate
        /etc/init.d/rsyslog reload > /dev/null
    endscript
}
```
2) Создаем файл nano /etc/logrotate.d/ssh_auth с содержимым 
```
/var/log/ssh_auth.log {
    missingok
    notifempty
    size 10M
    rotate 7
    postrotate
        /root/ssh_login_notify.sh
    endscript
}
```
3) Создаем файл nano ssh_login_notify.sh с содержимым
```
#!/bin/bash

LOGDIR="/var/log/"
BASELOG="ssh_auth.log"
TMPFILE="/tmp/ssh_auth.tmp"
ARCHIVE="ssh_auth.log.*"

# Создаем временный файл
touch "$TMPFILE"

# Обрабатываем текущий лог-файл и все архивные файлы
for LOGFILE in "$LOGDIR$BASELOG" $LOGDIR$ARCHIVE; do
    if [ -f "$LOGFILE" ]; then
        echo "Обработка файла: $LOGFILE" >> /var/log/ssh_login_notify.log

        # Сохраняем содержимое лог-файла в временный файл
        cat "$LOGFILE" >> "$TMPFILE"
    fi
done

# Находим последнюю строку с успешным входом
LAST_ENTRY=$(grep 'Accepted' "$TMPFILE" | tail -n 1)

if [ -z "$LAST_ENTRY" ]; then
    echo "Нет записей об успешных входах" >> /var/log/ssh_login_notify.log
    rm "$TMPFILE"
    exit 0
fi

# Извлекаем IP адрес
IP=$(echo "$LAST_ENTRY" | awk -F'from ' '{print $2}' | awk '{print $1}')

# Проверяем, корректный ли IP
if [ -z "$IP" ]; then
    echo "IP не найден. Входные данные: $LAST_ENTRY" >> /var/log/ssh_login_notify.log
    rm "$TMPFILE"
    exit 1
fi

# Кодируем текст для URL
TEXT=$(printf '%s' "Успешный вход с IP: $IP" | sed 's/ /%20/g' | sed 's/:/%3A/g')

# Отправляем сообщение в Telegram
RESPONSE=$(curl -s "https://api.telegram.org/bot{key:token}/sendMessage?chat_id={chat_id}&text=$TEXT")

# Логируем ответ от curl
echo "Ответ curl: $RESPONSE" >> /var/log/ssh_login_notify.log

# Удаляем временный файл
rm "$TMPFILE"

```
4) Откройте sudo nano /etc/ssh/sshd_config и выставляем
```
LogLevel VERBOSE
```
5) Создайте файл nano /etc/rsyslog.d/50-default.conf с содержимым
```
if $programname == 'sshd' and $msg contains 'Accepted' then /var/log/ssh_auth.log
& stop
```
6) Ребутаем сервисы
```
sudo systemctl restart rsyslog
sudo systemctl restart ssh

```
