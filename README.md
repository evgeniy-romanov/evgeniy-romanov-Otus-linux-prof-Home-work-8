# Инициализация системы. Systemd 

## Задачи
1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig).
2. Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi).
3. Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.

## 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig).
Создаем конфигурационный файл для сервиса:
```
[root@hw08-systemd ~] cat > /etc/sysconfig/watchlog
# Configuration file for my watchlog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```
Создаю лог-файл на случай, если ниже созданное задание в crontab отработает позже создаваемого сервиса:
```
[root@hw08-systemd ~] > /var/log/watchlog.log
```
Создаю скрипты для пополнения лога:
```
[root@hw08-systemd ~]# cat > /opt/alert_add.sh
#!/bin/bash
/bin/echo `/bin/date "+%b %d %T"` ALERT >> /var/log/watchlog.log

[root@hw08-systemd ~]# cat /opt/tail_add.sh
#!/bin/bash
/bin/tail /var/log/messages >> /var/log/watchlog.log

[root@hw07-systemd ~]# chmod +x /opt/tail_add.sh /opt/alert_add.sh
```


Создаю задания в crontab:
```
[root@hw08-systemd ~] crontab -e
*/3  *  *  *  * /opt/tail_add.sh
*/5  *  *  *  * /opt/alert_add.sh
```

Создаю непосредственно скрипт, который будет выполняться в ходе работы сервиса:
```
[root@hw08-systemd ~] cat > /opt/watchlog.sh
#!/bin/bash
WORD=$1
LOG=$2
DATE=`/bin/date`
if grep $WORD $LOG &> /dev/null; then
    logger "$DATE: I found word, Master!"
	exit 0
else
    exit 0
fi
```

Добавляю скрипту бит исполнения:
```
[root@hw08-systemd ~] chmod +x /opt/watchlog.sh
```
Формирую unit-файл сервиса:
```
[root@hw08-systemd ~] cat > /etc/systemd/system/watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Формирую unit-файл таймера:
```
[root@hw08-systemd ~] cat > /etc/systemd/system/watchlog.timer
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

Обновляю информацию о юнитах:
```
[root@hw08-systemd ~] systemctl daemon-reload
```

Запускаю юнит:
```
[root@hw08-systemd ~] systemctl start watchlog.timer
[root@hw08-systemd ~] systemctl enable watchlog.timer
```

Проверяю:
```
[root@hw08-systemd ~] tail -f /var/log/messages
un  5 16:17:12 localhost systemd: Created slice User Slice of vagrant.
Jun  5 16:17:12 localhost systemd: Started Session 12 of user vagrant.
Jun  5 16:17:12 localhost systemd-logind: New session 12 of user vagrant.
Jun  5 16:17:19 localhost su: (to root) vagrant on pts/0
Jun  5 16:17:33 localhost systemd: Starting My watchlog service...
Jun  5 16:17:33 localhost root: Sun Jun  5 16:17:33 UTC 2022: I found word, Master!
Jun  5 16:17:33 localhost systemd: Started My watchlog service.
Jun  5 16:18:01 localhost systemd: Created slice User Slice of root.
Jun  5 16:18:01 localhost systemd: Started Session 13 of user root.
Jun  5 16:18:01 localhost systemd: Removed slice User Slice of root.
Jun  5 16:18:03 localhost systemd: Starting My watchlog service...
Jun  5 16:18:03 localhost root: Sun Jun  5 16:18:03 UTC 2022: I found word, Master!
Jun  5 16:18:03 localhost systemd: Started My watchlog service.
Jun  5 16:18:33 localhost systemd: Starting My watchlog service...
Jun  5 16:18:33 localhost root: Sun Jun  5 16:18:33 UTC 2022: I found word, Master!
Jun  5 16:18:33 localhost systemd: Started My watchlog service.
Jun  5 16:19:03 localhost systemd: Starting My watchlog service...
Jun  5 16:19:03 localhost root: Sun Jun  5 16:19:03 UTC 2022: I found word, Master!
Jun  5 16:19:03 localhost systemd: Started My watchlog service.

```

## 2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.

Устанавливаем spawn-fcgi и необходимые для него пакеты:
```
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```
Раскомментировать строки с переменными в /etc/sysconfig/spawn-fcgi, должен принять следующий вид

![0.png](https://github.com/evgeniy-romanov/evgeniy-romanov-Otus-linux-prof-Home-work-8/blob/main/0.png)
Юнит файл имеет следующий вид ```vi /etc/systemd/system/spawn-fcgi.service```:
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
Убеждаемся, что все успешно работает:
```
systemctl start spawn-fcgi
systemctl status spawn-fcgi
```
![1.png](https://github.com/evgeniy-romanov/evgeniy-romanov-Otus-linux-prof-Home-work-8/blob/main/1.png)

## 3.Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.


Создаю первый юнит-файл:
```
[root@hw08-systemd ~]# cat > /etc/systemd/system/httpd@first.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
Создаю второй юнит-файл:
```
[root@hw08-systemd ~]# cat /etc/systemd/system/httpd@second.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Конфиги для первого и второго юнита:
```
[root@hw08-systemd ~]# grep -P "^OPTIONS" /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf

[root@hw08-systemd ~]# grep -P "^OPTIONS" /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
```

Настройки pid-файла и прослушиваемого порта в конфиге для первого и второго юнитов:
```
[root@hw08-systemd ~]# grep -P "^PidFile|^Listen" /etc/httpd/conf/first.conf
PidFile "/var/run/httpd-first.pid"
Listen 80
[root@hw08-systemd ~]# grep -P "^PidFile|^Listen" /etc/httpd/conf/second.conf
PidFile "/var/run/httpd-second.pid"
Listen 8080
```

Поочередно запускаем юниты:
```
[root@hw08-systemd ~]# systemctl start httpd@first
[root@hw08-systemd ~]# systemctl start httpd@second
```

Проверяем результат:
```
[root@hw08-systemd ~]# ss -ntlp | grep httpd
LISTEN     0      128       [::]:8080                  [::]:*                   users:(("httpd",pid=3483,fd=4),("httpd",pid=3482,fd=4),("httpd",pid=3481,fd=4),("httpd",pid=3480,fd=4),("httpd",pid=3479,fd=4),("httpd",pid=3478,fd=4))
LISTEN     0      128       [::]:80                    [::]:*                   users:(("httpd",pid=3476,fd=4),("httpd",pid=3475,fd=4),("httpd",pid=3474,fd=4),("httpd",pid=3473,fd=4),("httpd",pid=3472,fd=4),("httpd",pid=3471,fd=4))

```

