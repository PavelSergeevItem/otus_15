**Сбор и анализ логов**

**1. Описание задания**

1. В Vagrant разворачиваем 2 виртуальные машины web и log
2. на web настраиваем nginx
3. на log настраиваем центральный лог сервер на любой системе на выбор
journald;
rsyslog;
elk.
4. настраиваем аудит, следящий за изменением конфигов nginx
5. 
Все критичные логи с web должны собираться и локально и удаленно.
Все логи с nginx должны уходить на удаленный сервер (локально только критичные).
Логи аудита должны также уходить на удаленную систему.

 **2. Выполенение задания.**
1. Используя вагрантфайл из методички создал две виртуальные машины с хост именами WEB(192.168.56.10) и LOG(192.168.56.15).
2. Подключился через ssh к веб серверу, перешел в режим коренвого пользователя.
3. Для правильной работы с логами необходимо, чтобы на всех хостах должно быть настроено одинаковое время.
Указал часовой пояс: `cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime`.
Перезапустил службу NTP Chrony: `systemctl restart chronyd`.
Проверил статус службы:
```
[root@web ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2023-06-27 19:28:40 MSK; 6s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 22647 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 22643 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 22645 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─22645 /usr/sbin/chronyd
```
Проверил дату и время на сервере: 
```
[root@web ~]# date
Tue Jun 27 19:29:06 MSK 2023
```
Настроил NTP на обоих серверах.
4. На веб сервер установил nginx: `yum install -y nginx`.
Проверил статус сервера:
```
[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2023-06-27 19:34:49 MSK; 2s ago
  Process: 22842 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22840 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22839 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22844 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22844 nginx: master process /usr/sbin/nginx
           └─22846 nginx: worker process
```
5. Открыл новую вкладку в терминале, подключился через ssh к log серверу, перешел в рут режим.
   Проверил на сервере наличие rsyslog:
```
   [root@log audit]# yum list rsyslog
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.datahouse.ru
 * extras: mirrors.powernet.com.ru
 * updates: mirrors.datahouse.ru
Installed Packages
rsyslog.x86_64                                                                                 8.24.0-52.el7                                                                                      @anaconda
Available Packages
rsyslog.x86_64
```
Для того, чтоб лог-сервер мог принемать логи с удаленного сервера внес изенения в файл: `nano /etc/rsyslog.conf`
```
# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
```
В конце файл добавил строку: `#Add remote logs $template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log" *.* ?RemoteLogs`
Сохранил и перезапустил службу rsyslog: `systemctl restart rsyslog`.
Проверил, что нужные порты открыт на сервер:
```
[root@log ~]# ss -tuln
Netid State      Recv-Q Send-Q   Local Address:Port                  Peer Address:Port              
udp   UNCONN     0      0            127.0.0.1:323                              *:*                  
udp   UNCONN     0      0                    *:68                               *:*                  
udp   UNCONN     0      0                    *:111                              *:*                  
udp   UNCONN     0      0                    *:929                              *:*                  
udp   UNCONN     0      0                    *:514                              *:*                  
udp   UNCONN     0      0                [::1]:323                           [::]:*                  
udp   UNCONN     0      0                 [::]:111                           [::]:*                  
udp   UNCONN     0      0                 [::]:929                           [::]:*                  
udp   UNCONN     0      0                 [::]:514                           [::]:*                  
tcp   LISTEN     0      128                  *:111                              *:*                  
tcp   LISTEN     0      128                  *:22                               *:*                  
tcp   LISTEN     0      100          127.0.0.1:25                               *:*                  
tcp   LISTEN     0      25                   *:514                              *:*                  
tcp   LISTEN     0      128               [::]:111                           [::]:*                  
tcp   LISTEN     0      128               [::]:22                            [::]:*                  
tcp   LISTEN     0      100              [::1]:25                            [::]:*                  
tcp   LISTEN     0      25                [::]:514                           [::]:*
```
6. Опять перешел на веб серверл.
Отредактировал файл  `nano /etc/nginx/nginx.conf`, в разделе на сервера добавил строки:
```
 error_log  /var/log/nginx/error.log;
 error_log  syslog:server=192.168.50.15:514,tag=nginx_error;
 access_log syslog:server=192.168.50.15:514,tag=nginx_access,severity=info combined;
```
Прверил, что конфиг nginx указан правильно:
```
[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Перезапустил nginx.
Для проверки, что логи ошибок отправляются на удаленный сервер, удалил картирку, к которой будет обращаться nginx во время октрытия веб-страницы: `rm /usr/share/nginx/html/img/header-background.png`.
Окрыл стартовую страницу http://192.168.56.10 через барузер хозяйской ос. 
7. Далее перешел вного на лог-сервер и посмотрел информацию об nginx:
```
[root@log audit]# cat /var/log/rsyslog/web/nginx_access.log 
Jun 28 10:56:50 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:50 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:50 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:50 +0300] "GET /img/centos-logo.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:50 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:50 +0300] "GET /img/html-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:50 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:50 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:51 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:51 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:51 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:51 +0300] "GET /img/centos-logo.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:51 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:51 +0300] "GET /img/html-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 10:56:51 web nginx_access: 192.168.56.1 - - [28/Jun/2023:10:56:51 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 17:21:07 web nginx_access: 192.168.56.1 - - [28/Jun/2023:17:21:07 +0300] "GET /img/centos-logo.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 17:21:07 web nginx_access: 192.168.56.1 - - [28/Jun/2023:17:21:07 +0300] "GET /img/html-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 17:21:07 web nginx_access: 192.168.56.1 - - [28/Jun/2023:17:21:07 +0300] "GET /img/header-background.png HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
Jun 28 17:21:07 web nginx_access: 192.168.56.1 - - [28/Jun/2023:17:21:07 +0300] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/114.0"
```
```
[root@log audit]# cat /var/log/rsyslog/web/nginx_error.log 
Jun 28 10:56:50 web nginx_error: 2023/06/28 10:56:50 [error] 23872#23872: *1 open() "/usr/share/nginx/html/img/centos-logo.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/centos-logo.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Jun 28 10:56:50 web nginx_error: 2023/06/28 10:56:50 [error] 23872#23872: *2 open() "/usr/share/nginx/html/img/html-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/html-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Jun 28 10:56:50 web nginx_error: 2023/06/28 10:56:50 [error] 23872#23872: *3 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Jun 28 10:56:51 web nginx_error: 2023/06/28 10:56:51 [error] 23872#23872: *2 open() "/usr/share/nginx/html/img/centos-logo.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/centos-logo.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Jun 28 10:56:51 web nginx_error: 2023/06/28 10:56:51 [error] 23872#23872: *3 open() "/usr/share/nginx/html/img/html-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/html-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Jun 28 10:56:51 web nginx_error: 2023/06/28 10:56:51 [error] 23872#23872: *1 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
Jun 28 17:21:07 web nginx_error: 2023/06/28 17:21:07 [error] 24361#24361: *1 open() "/usr/share/nginx/html/img/centos-logo.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/centos-logo.png HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"
```
8. Настройка аудита, контролирующего изменения конфигурации nginx.
Проверил версию утилиту auditd.
```
[root@web nginx]# rpm -qa | grep audit
audit-2.8.5-4.el7.x86_64
audit-libs-2.8.5-4.el7.x86_64
```
Настроил аудит изменения конфигурации nginx для этого в конце файла /etc/audit/rules.d/audit.rules добавим следующие строки:
```
w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
```
Проверил работу аудита на веб-сервере:
```
[root@web nginx]# grep nginx_conf /var/log/audit/audit.log
type=CONFIG_CHANGE msg=audit(1687939189.167:1371): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1687939189.167:1372): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=SYSCALL msg=audit(1687939260.594:1376): arch=c000003e syscall=2 success=yes exit=3 a0=9a98a0 a1=441 a2=1b6 a3=63 items=2 ppid=22615 pid=23971 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="nano" exe="/usr/bin/nano" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
type=SYSCALL msg=audit(1687939276.703:1384): arch=c000003e syscall=2 success=yes exit=3 a0=9ae1d0 a1=241 a2=1b6 a3=7fff1d814420 items=2 ppid=22615 pid=23971 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="nano" exe="/usr/bin/nano" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CONFIG_CHANGE msg=audit(1687939663.123:1390): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687939663.123:1391): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687939663.123:1394): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687939663.123:1395): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SYSCALL msg=audit(1687940307.407:1397): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=1120420 a2=1ed a3=7ffdf28485a0 items=1 ppid=22615 pid=24134 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CONFIG_CHANGE msg=audit(1687940611.135:1402): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687940611.135:1403): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687940611.135:1406): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687940611.135:1407): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687941151.787:1413): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687941151.787:1414): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687941151.787:1417): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687941151.787:1418): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SYSCALL msg=audit(1687958224.279:1459): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=12a7420 a2=1a4 a3=7fff65f01620 items=1 ppid=22615 pid=24365 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CONFIG_CHANGE msg=audit(1687960921.698:1473): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.698:1474): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.706:1477): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.706:1478): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.143:1484): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.143:1485): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.146:1488): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.146:1489): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SYSCALL msg=audit(1687960988.656:1491): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=bde420 a2=1ed a3=7ffce08e77a0 items=1 ppid=22615 pid=24519 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=33 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
```
9. Настроил пересылку логов на удаленный сервер. Для этого уставновил пакет audispd-plugins: `yum -y install audispd-plugins`.
Нашел и поменаля в файле /etc/audit/auditd.conf
```
log_format = RAW
name_format = HOSTNAME
```
В файле /etc/audisp/plugins.d/au-remote.conf поменял параметр active на yes:
```
active = yes
direction = out
path = /sbin/audisp-remote
type = always
#args =
format = string
```
В файле /etc/audisp/audisp-remote.conf указал адрес сервера и порт, на который будут отправляться логи:
```
root@web ~]# cat /etc/audisp/audisp-remote.conf

remote_server = 192.168.56.15 
port = 60
```
Перезаупстил службу auditd: `service auditd restart`.
Перешл на лог-сервер и открыл порт TCP 60, для этого раскомментировал в файле etc/audit/auditd.conf строку `tcp_listen_port = 60`.
Перезапустил службу `service auditd restart`.
Для проверки работы пересылки логов, перешел на внось веб-сервер и внес изменения в файлах nginx:
```
[root@web nginx]# ls -l /etc/nginx/nginx.conf
-rwxr-xr-x. 1 root root 2495 Jun 27 11:01 /etc/nginx/nginx.conf
[root@web ~]# chmod +x /etc/nginx/nginx.conf
[root@web nginx]# ls -l /etc/nginx/nginx.conf
-rwxr-xr-x. 1 root root 2495 Jun 28 11:01 /etc/nginx/nginx.conf
```
Перешел на лог-сервер и проверил, что логи пересылаются:
```
grep web /var/log/audit/audit.log
node=web type=DAEMON_START msg=audit(1687960921.557:69): op=start ver=2.8.5 format=raw kernel=3.10.0-1127.el7.x86_64 auid=4294967295 pid=24446 uid=0 ses=4294967295 subj=system_u:system_r:auditd_t:s0 res=success
node=web type=CONFIG_CHANGE msg=audit(1687960921.698:1473): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.698:1474): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.706:1475): audit_backlog_limit=8192 old=8192 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.706:1476): audit_failure=1 old=1 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.706:1477): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960921.706:1478): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SERVICE_START msg=audit(1687960921.706:1479): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=auditd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
node=web type=DAEMON_END msg=audit(1687960940.969:70): op=terminate auid=1000 pid=24477 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 res=success
node=web type=DAEMON_START msg=audit(1687960942.009:5986): op=start ver=2.8.5 format=raw kernel=3.10.0-1127.el7.x86_64 auid=4294967295 pid=24494 uid=0 ses=4294967295 subj=system_u:system_r:auditd_t:s0 res=success
node=web type=CONFIG_CHANGE msg=audit(1687960942.143:1484): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.143:1485): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=remove_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.145:1486): audit_backlog_limit=8192 old=8192 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.145:1487): audit_failure=1 old=1 auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.146:1488): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1687960942.146:1489): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=SERVICE_START msg=audit(1687960942.153:1490): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=auditd comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=success'
```
Задание выполенно.


