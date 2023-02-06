Создаем файл watchlog в директории /etc/sysconfig/  с содержимым 
```
# Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

Создаем файл watchlog.log в директории /var/log/ с произвольным содержимым, но обязательно наличием слова ALERT (которое мы будем искать нашим сервисом)
прим
```
A "Hello, World!" program is generally a computer program that ignores any input and outputs or displays a message similar to "Hello, World!".
A small piece of ALERT code in most general-purpose programming languages, this program is used to illustrate a language's basic syntax.
"Hello, World!" programs are often the first a student learns to write in a given language,[1]
and they can also be used as a sanity check to ensure computer software intended to compile or run source code is correctly installed
and that its operator understands how to use it.
```
И сделаем скрипт, который logger'ом ищет слово
```
[root@otussystemd opt]# nano watchlog.sh
[root@otussystemd opt]# chmod u+x watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
    logger "$DATE: I found the word, master!"
else 
  exit 0
fi
```

Далее требуется создать юнит-файл, который будет запускать наш сервис watchlog
```
[root@otussystemd system]# cat <<'EOF' >>watchlog.service
> [Unit]
> Description=My watchlog service
> 
> [Service]
> Type=oneshot
> EnvironmentFile=/etc/sysconfig/watchlog
> ExecStart=/opt/watchlog.sh $WORD $LOG 
> EOF
```
И таймер, по которому будет запускаться сервис
```
[root@otussystemd system]# cat <<EOF >>watchlog.timer
> [Unit]
> Description=Run watchlog every 30 seconds
> 
> [Timer]
> #Run every 30 sec
> OnUnitActiveSec=30
> Unit=watchlog.service
> 
> [Install]
> WantedBy=multi-user.target
> EOF
```
Запускаем сервис и таймер
```
[root@otussystemd system]# systemctl start watchlog.timer
[root@otussystemd system]# systemctl start watchlog.service
```
Проверим
```
[root@otussystemd system]# tail -f /var/log/messages
Feb  6 19:22:01 localhost root: Mon Feb  6 19:22:01 UTC 2023: I found the word, master!
Feb  6 19:22:01 localhost systemd: Started My watchlog service.
Feb  6 19:23:01 localhost systemd: Starting My watchlog service...
Feb  6 19:23:01 localhost root: Mon Feb  6 19:23:01 UTC 2023: I found the word, master!
Feb  6 19:23:01 localhost systemd: Started My watchlog service.
Feb  6 19:24:01 localhost systemd: Starting My watchlog service...
Feb  6 19:24:01 localhost root: Mon Feb  6 19:24:01 UTC 2023: I found the word, master!
Feb  6 19:24:01 localhost systemd: Started My watchlog service.
Feb  6 19:24:41 localhost systemd: Starting My watchlog service...
Feb  6 19:24:41 localhost root: Mon Feb  6 19:24:41 UTC 2023: I found the word, master!
Feb  6 19:24:41 localhost systemd: Started My watchlog service.
Feb  6 19:25:01 localhost systemd: Starting My watchlog service...
Feb  6 19:25:01 localhost root: Mon Feb  6 19:25:01 UTC 2023: I found the word, master!
```

Дальше по заданию скачиваем пакеты и создаем юнит-файл
```
[root@otussystemd system]# yum install epel-release -y && yum install spawn-fcgi php php-cli
[root@otussystemd system]# nano ./spawn-fcgi.service
```
```
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```
В его настройках /etc/sysconfig/spawn-fcgi меняем строчки 
```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```
Запускаем его
```
[root@otussystemd conf]# systemctl start spawn-fcgi
```
```
[root@otussystemd conf]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2023-02-06 19:52:49 UTC; 1h 6min ago
 Main PID: 24828 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
```


Дальше по заданию надо сделать так, чтобы можно было запустить несколько инстансов веб-сервера apache

Сначала идем в /usr/lib/systemd/system и находим юнит-файл httpd.service, делаем из него httpd@.service, для возможности использовать его как шаблон.
И меняем его немного, добавляя аргумент в название сервиса
```
[Unit]
Description=The Apache HTTP Server
Wants=httpd-init.service

After=network.target remote-fs.target nss-lookup.target httpd-init.service

Documentation=man:httpd.service(8)

[Service]
Type=notify
Environment=LANG=C
EnvironmentFile=/etc/sysconfig/httpd-%I ---------- вот сюда добавляем переменную -%I 
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
# Send SIGWINCH for graceful stop
KillSignal=SIGWINCH
KillMode=mixed
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
Дальше делаем 2 конфиг файла в /etc/httpd/conf
```
[root@otussystemd conf]# mv httpd.conf first.conf
[root@otussystemd conf]# nano second.conf
```
в second.conf дописываем его собственный Pid и говорим слушать порт, отличный от первого инстанса (первый слушает 80й)
```
PidFile /var/run/httpd-second.pid
Listen 8080
```
Создаем в /etc/sysconfig конфиг файлы httpd-first и httpd-second
с содержанием таких опций:
```
OPTIONS=-f conf/first.conf
```
```
OPTIONS=-f conf/second.conf
```
для того, чтобы инстансы знали откуда вычитывать конфиг.


Всё, дальше запускаем
```
[root@otussystemd]# systemctl start httpd@first
```
```
[root@otussystemd]# systemctl start httpd@second
```
Проверка слушания портов обоими инстансами:
```
[root@otussystemd conf]# ss -tunlp | grep httpd
tcp    LISTEN     0      128    [::]:8080               [::]:*                   users:(("httpd",pid=26711,fd=4),("httpd",pid=26710,fd=4),("httpd",pid=26709,fd=4),("httpd",pid=26708,fd=4),("httpd",pid=26707,fd=4),("httpd",pid=26706,fd=4))
tcp    LISTEN     0      128    [::]:80                 [::]:*                   users:(("httpd",pid=26671,fd=4),("httpd",pid=26670,fd=4),("httpd",pid=26669,fd=4),("httpd",pid=26668,fd=4),("httpd",pid=26667,fd=4),("httpd",pid=26666,fd=4))
```
