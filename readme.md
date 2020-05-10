# Описание решения
### Конфигурация стенда
Ip|Servername|Роль|Установленное ПО|Порты
---|---|---|---|---
192.168.50.10|web|Web-сервер|Nginx|TCP 80 - порт nginx
192.168.50.11|log|Log-сервер|Rsyslog-сервер|UDP 514 - порт Rsyslog

Развертывание стенда происходит с помощью __Vagrant__ и __Ansible__.

### 1. Настройка сбора логов ОС
__Задание__: _все критичные логи с web должны собираться и локально и удаленно_.

1. На __web-сервер__ создан _/etc/rsyslog.d/logs_forwarding.conf_. В нем прописано правило перенаправления логов на __log-server__.
```
#*.crit @192.168.50.11:514
*.* @192.168.50.11:514
```
__Пояснение__. Строка отправки критичных логов закомментирована, поскольку мне было интересно перенаправить __все логи__ и на приемной стороне  "красиво" их обработать.

2. На __log-сервер__ создан _/etc/rsyslog.d/RemoteLogsCollection.conf_, где настроены правила приема и размещения логов:
    - Включен прием _syslog_ на _UDP_ порту _514_.
    - Прописан _template_, который будет складывать логи в папку с _ip_ источника по типу программы - автора лога.
    - Прописано правило фильтрации поступающих логов, согласно которому логи _nginx_ игнорируются (для них будет настроен отдельный сбор).
```
$ModLoad imudp
$UDPServerRun 514

template (name="RemoteLogs" type="string" string="/var/log/%fromhost-ip%/%PROGRAMNAME%.log")

if ($programname != 'nginx') then  {
action(type="omfile"    dynaFile="RemoteLogs")
}
```
__Проверка:__
1. Запускаем виртуальные машины:
```
# vagrant up
```
2. Выполняем provisioning для __web-сервер__ из папки с Vagrantfile:
```
# ansible-playbook -i inventories/staging/ playbooks/nginx_run_role.yml
```
3. Выполняем provisioning для __log-сервер__ из папки с Vagrantfile::
```
# ansible-playbook -i inventories/staging/ playbooks/logserver_run_role.yml
```
4. Заходим на __web-сервер__.
```
# vagrant ssh web
```
5. Заходим на __log-сервер__.
```
# vagrant ssh log
```
6. На __web-сервер__ что-нибудь делаем, например, перезагружаем любую службу:
```
#  sudo systemctl restart crond
```
7. На __log-сервер__ в  _/var/log/192.168.50.10_ видим актуальные журналы с __web-cервер__.

### 2. Настройка сбора логов NGINX
Задание: _все логи с nginx должны уходить на удаленный сервер (локально только критичные)_.
1. На __web-сервере__ в  _/etc/nginx/nginx.conf_ настроена конфигурация логирования _access_log_ и _error_log_ следующим образом:
```
error_log  /var/log/nginx/error.log crit;
error_log syslog:server=192.168.50.11:514;
access_log syslog:server=192.168.50.11:514;
```

2. На __log-сервер__  в  _/etc/rsyslog.d/RemoteLogsCollection.conf_ заданы фильтры и конфигурация шаблонов для приема _access.log_ и _error.log_. Логи будут складываться в папку с IP клиента:
```
template (name="nginx_access_template" type="string" string="/var/log/%fromhost-ip%/access.log")


template (name="nginx_error_template" type="string" string="/var/log/%fromhost-ip%/error.log")

if ($syslogfacility-text == 'local7' and $syslogseverity-text == 'info' and $programname == 'nginx') then  {
action(type="omfile"    dynaFile="nginx_access_template")
}

if ($syslogfacility-text == 'local7' and $syslogseverity-text == 'error' and $programname == 'nginx') then  {
action(type="omfile"    dynaFile="nginx_error_template")
}
```
__Проверка:__
1. Выполняем шаги 1-5 проверки предыдущего п.1, если это необходимо.
2. С любого сервера выполняем запрос к стартовой странице nginx:
```
# curl -v http://192.168.50.10
```

3.  С любого сервера выполняем запрос к несуществующей странице nginx:
```
# curl -v http://192.168.50.10/fake.html
```
4. На __log-сервер__ в _/var/log/192.168.50.10/nginx_ видим актуальные журналы nginx с __web-сервер__.
```
# cat /var/log/192.168.50.10/access.log
...
# cat /var/log/192.168.50.10/error.log
```
7. На  __web-сервер__ (192.168.50.11)  ничего не логируется :
```
# ll /var/log/nginx
```
(...кроме критичных. Не догадался как искусственно создать критичную ошибку для проверки, но верю, что все должно работать).

### 3. Настройка аудита и сбор логов аудита.
Задание: _логи аудита должны также уходить на удаленную систему_.
1. На __web-сервере__ выполнена настройка правила аудита файла _/etc/nginx/nginx.conf_ на запись. Настройка записана в   _/etc/audit/rules.d/audit.rules_. Проверить активные правила можно так:

```
[root@web vagrant]# auditctl -l
-w /etc/nginx/nginx.conf -p wa -k nginx_conf_audit
```
2. SElinux отключен совсем. Правильнее было бы написать правило, но победила лень-).


__Проверка:__
1. Выполняем шаги 1-5 проверки предыдущего п.1, если это необходимо.
2. На __web-сервере__ выполняем произвольную запись в аудируемый _/etc/nginx/nginx.conf_ :
```
echo '# Just comment for audit checking' >>  /etc/nginx/nginx.conf
```
3. На __log-сервере__ смотрим результат:
```
tail -f /var/log/messages | grep nginx_conf_audit
```


