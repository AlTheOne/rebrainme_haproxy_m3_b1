# Выполнение итогового задания

Мини практикум: HAProxy
Модуль: 03
Блок: 01

# Отчётность

См. директории репозитория

# Решение

## Настройка VM Master

### Настройка sysctl

Редактируем
```sh
sudo nano /etc/sysctl.conf
```

Содержимое
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 1
net.ipv4.conf.all.arp_filter = 0
net.ipv4.conf.<NETWORK_INTERFACE_ethX>.arp_filter = 1 
```

*Чтобы указать `NETWORK_INTERFACE_ethX` необходимо выполнить `ip a` и найти интерфейс для сети (например, `eth0`).*

Применить настройки
```sh
sudo sysctl -p
```

### Создать сетевой интерфейс

```sh
ip a
```

Нас интересует интерфейс `ethX:10`

Ищем примерно такую строку:
```
inet <EXTERNAL_IP_VM_MASTER>/XX brd XXX.XXX.XXX.XXX scope global eth0:10
```

Если нет, то создаём вручную
```sh
sudo ifconfig eth0:10 <EXTERNAL_IP_VM_MASTER> up
```

*Мне понадобилось предварительно установить net-tools: `sudo apt install net-tools -y`*


### Подготовка SSL сертификата

Генерация самоподписанного сертификата
```sh
openssl req -nodes -x509 -newkey rsa:2048 -keyout rebrain.key -out rebrain.crt -days 30
```

Сборка в один файл для HAProxy
```sh
cat rebrain.key rebrain.crt > rebrain.pem
```

Переместить по заданию
```sh
sudo cp rebrain.pem /etc/haproxy/rebrain.pem
```

### Настройка HAProxy

Открыть файл настроек
```sh
sudo nano /etc/haproxy/haproxy.cfg
```

Содержимое файла
```
defaults

  # Добавить строчку
  log-format {\"type\":\ \"haproxy\",\"client_ip\":\ \"%ci\",\"client_port\":\ \"%cp\",\"request_date\":\ \"[%t]\",\"frontend_name\":\ \"%f\",\"backend_name\":\ \"%b\",\"server_name\":\ \"%s\",\"http_status_code\":\ \"%ST\",\"bytes_read\":\ \"%B\",\"termination_state\":\ \"%ts\",\"actconn\":\ \"%ac\",\"feconn\":\ \"%fc\",\"beconn\":\ \"%bc\",\"srv_conn\":\ \"%sc\",\"retries\":\ \"%rc\",\"server_queue/backend_queue\":\ \"%sq/%bq\",\"http_method\":\ \"%HM\",\"http_request_and_version\":\ \"%HU\ %HV\",\"backend_server_ip\":\ \"%si\",\"backend_server_port\":\ \"%sp\",\"TR\":\ \"%TR\",\"Tw\":\ \"%Tw\",\"Tc\":\ \"%Tc\",\"Tr\":\ \"%Tr\",\"Ta\":\ \"%Ta\"}


cache rebrain_cache
  total-max-size 128
  max-object-size 10000
  max-age 30


listen stat
  bind *:777
  mode http
  stats enable
  stats uri /stats
  stats show-legends
  stats refresh 30
  stats auth admin:admin
  stats hide-version
  stats realm Haproxy\ Statistics


frontend rebrain_front
  bind *:443 ssl crt /etc/haproxy/rebrain.pem
  mode http

  http-request set-header X-Forwarded-For %[src]

  http-request cache-use rebrain_cache
  http-response cache-store rebrain_cache

  acl url_api path_beg -i /api
  acl url_lk path_beg -i /lk

  use_backend rebrain_api if url_api
  use_backend rebrain_lk if url_lk
  default_backend rebrain_back


frontend front_sql
  bind *:3307
  mode tcp
  option tcplog
  default_backend rebrain_sql


backend rebrain_api
  mode http
  balance roundrobin
  cookie REBRAIN indirect nocache insert
  server rebrain_01_80 127.0.0.1:80 check cookie rebrain_01_80
  server rebrain_02_80 127.0.0.1:80 check cookie rebrain_02_80
  option prefer-last-server


backend rebrain_lk
  mode http
  balance leastconn
  acl is_cached path_end .js .php .css
  http-request cache-use rebrain_cache if is_cached
  http-response cache-store rebrain_cache if is_cached
  server rebrain_01_81 127.0.0.1:81 check inter 4s
  server rebrain_02_81 127.0.0.1:81 check inter 4s maxconn 80


backend rebrain_back
  mode http
  balance source
  cookie PHPSESSID prefix nocache
  server rebrain_01_82 127.0.0.1:82 check port 82 inter 8s maxconn 1100 cookie s1
  server rebrain_02_82 127.0.0.1:82 check port 82 inter 8s maxconn 1100 cookie s2
  server rebrain_03_82 127.0.0.1:82 check port 82 inter 8s maxconn 1100 cookie s3


backend rebrain_sql
  balance roundrobin
  mode tcp
  option mysql-check user haproxy
  server rebrain_db_1 127.0.0.1:3306 check port 3306 inter 2s rise 1 fall 2 maxconn 100
  server rebrain_db_2 127.0.0.1:3306 check port 3306 inter 2s rise 1 fall 2 maxconn 100

```

*Нужно убедиться, что имеется настройка в global (если нет - добавить): 
`tune.ssl.default-dh-param 2048`*

Проверка настроек
```sh
haproxy -f /etc/haproxy/haproxy.cfg -c
```

Запуск службы
```sh
sudo systemctl start haproxy.service
```

Статус
```sh
sudo systemctl status haproxy.service
```

Автозагрузка службы
```sh
sudo systemctl enable haproxy.service
```

### Установка keepalived

```sh
sudo apt update
```

Установка
```sh
sudo apt install keepalived -y
```

### Настройка keepalived

```sh
sudo nano /etc/keepalived/keepalived.conf
```

Содержимое
```
global_defs {
    notification_email {
        altheone.social@gmail.com
        smtp_connect_timeout 30
        enable_traps
    }
}

vrrp_script haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VRRP1 {
    state MASTER
    interface eth0
    virtual_router_id 69
    priority 49
    advert_int 1
    garp_master_delay 10
    debug 1
    authentication {
        auth_type PASS
        auth_pass 1066
    }

    unicast_src_ip <INTERNAL_IP_VM_MASTER>
    unicast_peer {
        <INTERNAL_IP_VM_BACKUP>
    }

    virtual_ipaddress {
        <EXTERNAL_IP_VM_MASTER>/XX brd XXX.XXX.XXX.XXX scope global label eth0:10
    }

    track_script {
        haproxy
    }

}
```

Запускаем keepalived
```sh
sudo systemctl start keepalived
```

Проверка состояния
```sh
sudo systemctl start keepalived
```

Включить автозагрузку
```sh
sudo systemctl enable keepalived
```

### Настройка RSyslog

Открыть файл
```sh
sudo nano /etc/rsyslog.conf
```

Изменения в файле
```

# Подключение модуля для UDP (используется устройствами Cisco и другими серверами)
module(load="imudp")
input(type="imudp" port="514")

# Подключение модуля для TCP (если необходимо для других сетевых устройств)
module(load="imtcp")
input(type="imtcp" port="514")

if $msg contains "{\"type\": \"haproxy\"" then /var/log/haproxy.json
```

Удалить файл, который автоматически был создан:
```sh
sudo rm /etc/rsyslog.d/49-haproxy.conf
```

```sh
sudo systemctl restart rsyslog
```


### Установка и настройка Prometheus

#### Установка

```sh
wget https://github.com/prometheus/prometheus/releases/download/v2.26.0/prometheus-2.26.0.linux-amd64.tar.gz
```

```sh
tar zxf prometheus-2.26.0.linux-amd64.tar.gz
mv prometheus-2.26.0.linux-amd64 prometheus
```

```sh
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

```sh
sudo mv prometheus/prometheus /usr/local/bin/
sudo mv prometheus/promtool /usr/local/bin/
sudo mv prometheus/prometheus.yml /etc/prometheus/
```

```sh
sudo useradd prometheus
```

```sh
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Изменить настройки
```sh
sudo nano /etc/prometheus/prometheus.yml
```

Содержимое файла
```
scrape_configs:

  # Дополнить этим...
  - job_name: 'haproxy'
    static_configs:
    - targets: ['localhost:9091']
```

#### Подготовка service

```sh
sudo nano /etc/systemd/system/prometheus.service
```

Вставляем:
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/data

[Install]
WantedBy=default.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

### Установка и настройка HAProxy Exporter

Установка
```sh
wget https://github.com/prometheus/haproxy_exporter/releases/download/v0.8.0/haproxy_exporter-0.8.0.linux-amd64.tar.gz
tar -xzf haproxy_exporter-0.8.0.linux-amd64.tar.gz
```

```sh
sudo cp haproxy_exporter-0.8.0.linux-amd64/haproxy_exporter /usr/bin/
```

```sh
sudo nano /etc/systemd/system/haproxy_exporter.service
```

Содержимое файла
```
[Unit]
Description=Haproxy_exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=haproxy
Group=haproxy
Restart=on-failure
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/bin/haproxy_exporter \
  --haproxy.scrape-uri="http://admin:admin@localhost:777/stats;csv" \
  --web.listen-address=0.0.0.0:9091
SyslogIdentifier=haproxy_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable haproxy_exporter
sudo systemctl start haproxy_exporter
sudo systemctl status haproxy_exporter
```


### Установка Grafana

Т.к. существуют проблемы с доступом из РФ, то устанавливаем через докер:
```sh
sudo docker run -d --network=host --name=grafana -p 3000:3000 grafana/grafana
```

*Важно указать `--network=host`*

## Настройка MySQL

Открыть CLI без пароля от root
```sh
sudo mysql -u root
```

Создать учётную запись
```sql
CREATE USER 'haproxy'@'127.0.0.1' IDENTIFIED WITH mysql_native_password BY '';
GRANT ALL PRIVILEGES ON *.* TO 'haproxy'@'127.0.0.1';
FLUSH PRIVILEGES;
```

*Использование модуля `mysql_native_password` важно, т.к. по умолчанию с версии >=v8 используется  `caching_sha2_password` и возникает ошибка авторизации.*
Подробнее: https://stackoverflow.com/a/56509065

### Запуск приложений

```sh
sudo docker start $(sudo docker ps -qa)
```


## Настройка VM BACKUP

### Настройка sysctl

Редактируем
```sh
sudo nano /etc/sysctl.conf
```

Содержимое
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 1
net.ipv4.conf.all.arp_filter = 0
net.ipv4.conf.<NETWORK_INTERFACE_ethX>.arp_filter = 1 
```

*Чтобы указать `NETWORK_INTERFACE_ethX` необходимо выполнить `ip a` и найти интерфейс для сети (например, `eth0`).*

Применить настройки
```sh
sudo sysctl -p
```

### Подготовка SSL сертификата

Копируем с **VM Master**
```sh
scp user@<EXTERNAL_IP_VM_MASTER>:/home/user/rebrain.pem /home/user/rebrain.pem
```

```sh
sudo mv rebrain.pem /etc/haproxy/
```

### Настройка HAProxy

См. настройку на **VM Master**

### Установка keepalived

```sh
sudo apt update
```

Установка
```sh
sudo apt install keepalived -y
```

### Настройка keepalived

```sh
sudo nano /etc/keepalived/keepalived.conf
```

Содержимое
```
global_defs {
    notification_email {
        altheone.social@gmail.com
        smtp_connect_timeout 30
        enable_traps
    }
}

vrrp_script haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
}

vrrp_instance VRRP1 {
    state BACKUP
    interface eth0
    virtual_router_id 69
    priority 48
    advert_int 1
    garp_master_delay 10
    debug 1
    authentication {
        auth_type PASS
        auth_pass 1066
    }

    unicast_src_ip <INTERNAL_IP_VM_BACKUP>
    unicast_peer {
        <INTERNAL_IP_VM_MASTER>
    }

    virtual_ipaddress {
        <EXTERNAL_IP_VM_MASTER>/XX brd XXX.XXX.XXX.XXX scope global label eth0:10
    }

    track_script {
        haproxy
    }

}
```

Запускаем keepalived
```sh
sudo systemctl start keepalived
```

Включить автозагрузку
```sh
sudo systemctl enable keepalived
```


### Настройка RSyslog

Открыть файл
```sh
sudo nano /etc/rsyslog.conf
```

Изменения в файле
```

# Подключение модуля для UDP (используется устройствами Cisco и другими серверами)
module(load="imudp")
input(type="imudp" port="514")

# Подключение модуля для TCP (если необходимо для других сетевых устройств)
module(load="imtcp")
input(type="imtcp" port="514")

if $msg contains "{\"type\": \"haproxy\"" then /var/log/haproxy.json
```

Удалить файл, который автоматически был создан:
```sh
sudo rm /etc/rsyslog.d/49-haproxy.conf
```

```sh
sudo systemctl restart rsyslog
```


### Установка и настройка Prometheus

#### Установка

```sh
wget https://github.com/prometheus/prometheus/releases/download/v2.26.0/prometheus-2.26.0.linux-amd64.tar.gz
```

```sh
tar zxf prometheus-2.26.0.linux-amd64.tar.gz
mv prometheus-2.26.0.linux-amd64 prometheus
```

```sh
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

```sh
sudo mv prometheus/prometheus /usr/local/bin/
sudo mv prometheus/promtool /usr/local/bin/
sudo mv prometheus/prometheus.yml /etc/prometheus/
```

```sh
sudo useradd prometheus
```

```sh
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Изменить настройки
```sh
sudo nano /etc/prometheus/prometheus.yml
```

Содержимое файла
```
scrape_configs:

  # Дополнить этим...
  - job_name: 'haproxy'
    static_configs:
    - targets: ['localhost:9091']
```

#### Подготовка Prometheus service

```sh
sudo nano /etc/systemd/system/prometheus.service
```

Вставляем:
```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/data

[Install]
WantedBy=default.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```


### Установка и настройка HAProxy Exporter

Установка
```sh
wget https://github.com/prometheus/haproxy_exporter/releases/download/v0.8.0/haproxy_exporter-0.8.0.linux-amd64.tar.gz
tar -xzf haproxy_exporter-0.8.0.linux-amd64.tar.gz
```

```sh
sudo cp haproxy_exporter-0.8.0.linux-amd64/haproxy_exporter /usr/bin/
```

```sh
sudo nano /etc/systemd/system/haproxy_exporter.service
```

Содержимое файла
```
[Unit]
Description=Haproxy_exporter
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=haproxy
Group=haproxy
Restart=on-failure
ExecReload=/bin/kill -HUP $MAINPID
ExecStart=/usr/bin/haproxy_exporter \
  --haproxy.scrape-uri="http://admin:admin@localhost:777/stats;csv" \
  --web.listen-address=0.0.0.0:9091
SyslogIdentifier=haproxy_exporter
Restart=always

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl daemon-reload
sudo systemctl enable haproxy_exporter
sudo systemctl start haproxy_exporter
sudo systemctl status haproxy_exporter
```


## Настройка MySQL

Открыть CLI без пароля от root
```sh
sudo mysql -u root
```

Создать учётную запись
```sql
CREATE USER 'haproxy'@'127.0.0.1' IDENTIFIED WITH mysql_native_password BY '';
GRANT ALL PRIVILEGES ON *.* TO 'haproxy'@'127.0.0.1';
FLUSH PRIVILEGES;
```

*Использование модуля `mysql_native_password` важно, т.к. по умолчанию с версии >=v8 используется  `caching_sha2_password` и возникает ошибка авторизации.*
Подробнее: https://stackoverflow.com/a/56509065

### Запуск приложений

```sh
sudo docker start $(sudo docker ps -qa)
```
