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
