# Описание диплома

**Задача:** собрать отказоусточивый кластер Patroni, в котором:
* Все компоненты работают в отказоустойчивом режиме и нет единой точки отказа.
* Все комнонента добавлены в систему Observability и мониторятся.

Используемое ПО: 
* Ubuntu 24.04.3 LTS
* Prometheus 3.5.0
* Grafana 12.1.1
* Node Exporter 1.9.1
* ETCD 3.6.4

## Первичная подготовка

Устанавливаем ОС на 4 ВМ:
* Для кластера Patroni: ptrn-1, ptrn-2, ptrn-3
* Для системы Observability: prmt-1

Запускаем на каждой ВМ:

    sudo apt update && sudo apt upgrade -y && sudo apt reboot

Добавляем в /etc/hosts на всех машинах:

    192.168.2.96  ptrn-1
    192.168.2.91  ptrn-2
    192.168.2.98  ptrn-3
    192.168.2.99  prmt-1


## Установка Node Exporter

Выполняем на всех 4 нодах.

Команды скачивания бинарника и копирования его в /usr/local/bin:

    wget https://github.com/prometheus/node_exporter/releases/download/v1.9.1/node_exporter-1.9.1.linux-amd64.tar.gz
    tar -xvf node_exporter-1.9.1.linux-amd64.tar.gz
    mv node_exporter-1.9.1.linux-amd64/node_exporter /usr/local/bin/
    sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

Создаем systemd юнит:

    sudo tee /etc/systemd/system/node_exporter.service <<"EOF"
    [Unit]
    Description=Node Exporter

    [Service]
    User=node_exporter
    Group=node_exporter
    EnvironmentFile=-/etc/sysconfig/node_exporter
    ExecStart=/usr/local/bin/node_exporter $OPTIONS \
            --collector.systemd \
            --collector.processes

    [Install]
    WantedBy=multi-user.target
    EOF

Запускаем сервис и добавляем его в enabled:

    sudo systemctl daemon-reload && sudo systemctl start node_exporter && sudo systemctl status node_exporter && sudo systemctl enable node_exporter

Проверяем, что все 4 ВМ начинают отдавать метрики на порту 9100:

![Node Exported работает!](images/01-ne_metrics.png)

## Установка Prometheus и Grafana

Создаем на prmt-01 директории и меняем на них права:

    mkdir /opt/prometheus
    mkdir /opt/prometheus/rules
    mkdir /opt/prometheus_data
    mkdir /opt/grafana_data
    chown -R vasya:vasya /opt/prometheus
    chown -R vasya:vasya /opt/prometheus_data/
    chown -R vasya:vasya /opt/grafana_data/

Здесь vasya - пользователь, который был создан при установке Ubuntu и имеет id 1000:

    root@prmt-1:~# cat /etc/passwd | grep 1000
    vasya:x:1000:1000:vasya:/home/vasya:/bin/bash
    root@prmt-1:~# cat /etc/group | grep 1000
    vasya:x:1000:

Создаем файл **compose.yaml**:

    services:

    prometheus:
        image: prom/prometheus:v3.5.0
        container_name: prometheus
        user: "1000:1000"
        ports:
        - "9090:9090"
        volumes:
        - /opt/prometheus:/etc/prometheus
        - /opt/prometheus_data:/prometheus
        environment:
        - TZ=Europe/Moscow
        command:
        - '--config.file=/etc/prometheus/prometheus.yml'
        - '--storage.tsdb.path=/prometheus'
        networks:
        - monitoring_network
        restart: unless-stopped

    grafana:
        image: grafana/grafana:12.1.1
        container_name: grafana
        user: "1000:1000"
        ports:
        - "3000:3000"
        volumes:
        - /opt/grafana_data:/var/lib/grafana
        environment:
        - GF_SECURITY_ADMIN_USER=admin
        - GF_SECURITY_ADMIN_PASSWORD=admin123
        - GF_USERS_ALLOW_SIGN_UP=false
        - TZ=Europe/Moscow
        networks:
        - monitoring_network
        restart: unless-stopped

    networks:
    monitoring_network:

Проверяем, что контейнеры запустились:

    root@prmt-1:~# docker ps
    CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                                         NAMES
    5116461baacc   grafana/grafana:12.1.1   "/run.sh"                7 seconds ago   Up 7 seconds   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp   grafana
    4c2be41e9fdf   prom/prometheus:v3.5.0   "/bin/prometheus --c…"   12 hours ago    Up 11 hours    0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp   prometheus

## Настройка Prometheus

Копируем файл node-exporter.yml в директорию /opt/prometheus/rules и добавляем его в prometheus.yml. Добавляем эндпойнты в scrape_config:

    root@prmt-1:~# cat /opt/prometheus/prometheus.yml
    global:
    scrape_interval: 15s

    rule_files:
    - "rules/node-exporter.yml"

    scrape_configs:
    - job_name: 'prometheus'
        static_configs:
        - targets: ['localhost:9090']

    - job_name: 'node_exporter'
        static_configs:
        - targets: ['ptrn-1:9100', 'ptrn-2:9100', 'ptrn-3:9100', 'prmt-1:9100']

Проверяем что Prometheus работает на порту 9090 и успешно скрэйпит эндпойнты:

![Prometheus работает!](images/01-prometheus.png)

## Настройка Grafana

Добавляем в качестве Data Source наш сервер Prometheus:

![Prometheus как Data Source в Grafana](images/03-grafana_prom_ds.png)

Импортируем дашборд Node Exporter и проверяем, что данные собираются:

![Дашборд Node Exporter](images/04-ne_dashboard.png)

## Настройка алертов Grafana в Telegram

Создаем в Графане Contact Point с заданным ChatID и токеном:

![Contact Point](images/05-tg_contact_point.png)

Проверяем, что алерты приходят:

![Пример алерта в Телеге](images/06-tg_alert_example.png)

> NOTE: рендеринг красивых Go-шаблонов для алертов в Телеге останется за пределами данного дипломного проекта!

## Установка и настройка кластера ETCD

Выполняем на нодах ptrn-1, ptrn-2, ptrn-3:

    wget https://github.com/etcd-io/etcd/releases/download/v3.6.4/etcd-v3.6.4-linux-amd64.tar.gz
    tar xzvf etcd-v3.6.4-linux-amd64.tar.gz
    mv /tmp/etcd-v3.6.4-linux-amd64/etcd* /usr/local/bin/
    groupadd --system etcd
    useradd -s /sbin/nologin --system -g etcd etcd
    mkdir /opt/etcd
    mkdir /etc/etcd
    chown -R etcd:etcd /opt/etcd
    chmod -R 700 /opt/etcd/

Создаем конфигурационный файл /etc/etcd/etcd.conf.

ptrn-1:

    ETCD_NAME="ptrn-1"
    ETCD_LISTEN_CLIENT_URLS="http://192.168.2.96:2379,http://127.0.0.1:2379"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.2.96:2379"
    ETCD_LISTEN_PEER_URLS="http://192.168.2.96:2380"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.2.96:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
    ETCD_INITIAL_CLUSTER="ptrn-1=http://192.168.2.96:2380,ptrn-2=http://192.168.2.91:2380,ptrn-3=http://192.168.2.98:2380"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_DATA_DIR="/opt/etcd"
    ETCD_ELECTION_TIMEOUT="10000"
    ETCD_HEARTBEAT_INTERVAL="2000"
    ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
    ETCD_ENABLE_V2="true"

ptrn-2:

    ETCD_NAME="ptrn-2"
    ETCD_LISTEN_CLIENT_URLS="http://192.168.2.91:2379,http://127.0.0.1:2379"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.2.91:2379"
    ETCD_LISTEN_PEER_URLS="http://192.168.2.91:2380"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.2.91:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
    ETCD_INITIAL_CLUSTER="ptrn-1=http://192.168.2.96:2380,ptrn-2=http://192.168.2.91:2380,ptrn-3=http://192.168.2.98:2380"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_DATA_DIR="/opt/etcd"
    ETCD_ELECTION_TIMEOUT="10000"
    ETCD_HEARTBEAT_INTERVAL="2000"
    ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
    ETCD_ENABLE_V2="true"

ptrn-3:

    ETCD_NAME="ptrn-3"
    ETCD_LISTEN_CLIENT_URLS="http://192.168.2.98:2379,http://127.0.0.1:2379"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.2.98:2379"
    ETCD_LISTEN_PEER_URLS="http://192.168.2.98:2380"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.2.98:2380"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
    ETCD_INITIAL_CLUSTER="ptrn-1=http://192.168.2.96:2380,ptrn-2=http://192.168.2.91:2380,ptrn-3=http://192.168.2.98:2380"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_DATA_DIR="/opt/etcd"
    ETCD_ELECTION_TIMEOUT="10000"
    ETCD_HEARTBEAT_INTERVAL="2000"
    ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
    ETCD_ENABLE_V2="true"

Создаем на всех трёх нодах файл systemd-юнита:

    cat /etc/systemd/system/etcd.service
    [Unit]
    Description=Etcd Server
    Documentation=https://github.com/etcd-io/etcd
    After=network.target
    After=network-online.target
    Wants=network-online.target

    [Service]
    User=etcd
    Type=notify
    WorkingDirectory=/opt/etcd/
    EnvironmentFile=-/etc/etcd/etcd.conf
    User=etcd
    # set GOMAXPROCS to number of processors
    ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd"
    Restart=on-failure
    LimitNOFILE=65536
    IOSchedulingClass=realtime
    IOSchedulingPriority=0
    Nice=-20

    [Install]
    WantedBy=multi-user.target

Запускаем службу на всех трех нодах:

    systemctl daemon-reload && systemctl enable etcd && systemctl start etcd

Видим, что кластер собрался и работает:

    root@ptrn-2:~# etcdctl member list
    513eb46eca501, started, ptrn-1, http://192.168.2.105:2380, http://192.168.2.96:2379, false
    455812df4c39a5e5, started, ptrn-3, http://192.168.2.98:2380, http://192.168.2.98:2379, false
    83b8164138628079, started, ptrn-2, http://192.168.2.91:2380, http://192.168.2.91:2379, false

    root@ptrn-2:~# etcdctl endpoint status --cluster
    http://192.168.2.96:2379, 513eb46eca501, 3.6.4, 3.6.0, 20 kB, 16 kB, 20%, 0 B, false, false, 7, 26, 26, , , false
    http://192.168.2.98:2379, 455812df4c39a5e5, 3.6.4, 3.6.0, 20 kB, 16 kB, 20%, 0 B, true, false, 7, 26, 26, , , false
    http://192.168.2.91:2379, 83b8164138628079, 3.6.4, 3.6.0, 20 kB, 16 kB, 20%, 0 B, false, false, 7, 26, 26, , , false

    root@ptrn-2:~# etcdctl endpoint health --cluster
    http://192.168.2.98:2379 is healthy: successfully committed proposal: took = 48.877664ms
    http://192.168.2.91:2379 is healthy: successfully committed proposal: took = 48.710623ms
    http://192.168.2.96:2379 is healthy: successfully committed proposal: took = 48.770278ms

На prmt-1 добавляем метрики etcd в prometheus.yml:

    root@prmt-1:~# cat /opt/prometheus/prometheus.yml
    global:
    scrape_interval: 15s

    rule_files:
    - "rules/node-exporter.yml"

    scrape_configs:
    - job_name: 'prometheus'
        static_configs:
        - targets: ['localhost:9090']

    - job_name: 'node_exporter'
        static_configs:
        - targets: ['ptrn-1:9100', 'ptrn-2:9100', 'ptrn-3:9100', 'prmt-1:9100']

    - job_name: "etcd"
        scrape_interval: 15s
        metrics_path: /metrics
        static_configs:
        - targets: ["ptrn-1:2379", "ptrn-2:2379", "ptrn-3:2379"]

Импортируем дашборд в Графану и в ней наблюдаем как при перезагрузке лидера кластера статус лидера переходит к другой ноде:

![Дашборд ETCD](images/06-tg_alert_example.png)

Установка и настройка кластера ETCD завершена!

