ДЗ #1:
1. Установил Ubuntu Server 24.04, Docker, Docker Compose

2. Составил docker-compose.yml-файл следующего содержания:

services:
  wordpress:
    image: wordpress:php8.3-fpm
    container_name: wordpress
    volumes:
      - wordpress_data:/var/www/html
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wpdb
    depends_on:
      - mysql

  mysql:
    image: mysql:8.0
    container_name: mysql
    volumes:
      - db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wpdb
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass

  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /etc/docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - wordpress_data:/var/www/html
    depends_on:
      - wordpress

  prometheus:
    image: prom/prometheus:v2.53.4
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/var/lib/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/var/lib/prometheus'
      - '--web.enable-lifecycle'

  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    ports:
      - "9100:9100"

  blackbox:
    image: prom/blackbox-exporter
    container_name: blackbox-exporter
    ports:
      - "9115:9115"
    volumes:
      - /etc/docker/monitoring/blackbox.yml:/etc/blackbox_exporter/config.yml
    restart: unless-stopped

  php-fpm-exporter:
    image: hipages/php-fpm_exporter
    container_name: php-fpm-exporter
    ports:
      - "9253:9253"
    command:
      - "--phpfpm.scrape-uri=tcp://wordpress:9000/status"

  mysqld-exporter:
    image: prom/mysqld-exporter
    container_name: mysql-exporter
    restart: unless-stopped
    environment:
      - DATA_SOURCE_NAME="exporter:exporter_password@(mysql:3306)/"
    volumes:
      - /etc/docker/monitoring/.my.cnf:/etc/mysqld-exporter/.my.cnf
    ports:
      - "9104:9104"
    depends_on:
      - mysql
    command:
      - "--config.my-cnf=/etc/mysqld-exporter/.my.cnf"
      - "--mysqld.address=mysql:3306"

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter
    container_name: nginx-exporter
    ports:
      - "9113:9113"
    command:
      - "-nginx.scrape-uri=http://nginx:8080/nginx_status"

volumes:
  wordpress_data:
  db_data:
  prometheus_data:

3. Создал /etc/prometheus/prometheus.yml с необходимами job'ами

4. Установил и запустил через sudo docker-compose up -d контейнеры

5. Через браузер проинициализировал установку Wordpress CMS

ДЗ #2:
1. Добавил в docker-compose.yml параметры развёртывания VictoriaMetrics:

  victoria-metrics:
    image: victoriametrics/victoria-metrics:latest
    container_name: victoria-metrics
    ports:
      - "8428:8428"
    volumes:
      - ./victoria-metrics_data:/var/lib/victoria-metrics
    command:
      - -retentionPeriod=2w
      - -envflag.enable=true
      - -selfScrapeInterval=10s
      - -storageDataPath=/var/lib/victoria-metrics
      - -httpListenAddr=:8428

2. В prometheus.yml добавил подключение VictoriaMetrics:

remote_write:
  - url: http://victoria-metrics:8428/api/v1/write
    queue_config:
      max_samples_per_send: 10000
      capacity: 100000
      max_shards: 10

а также добавил лейбл site: prod

  external_labels:
    site: prod

ДЗ #3:
1. Добавил в docker-compose.yml параметры развёртывания AlertManager:

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - /etc/docker/monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - ./alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped
    depends_on:
      - prometheus

2. В prometheus.yml добавил подключение AlertManager:

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/alerts.yml

3. В /etc/prometheus/alerts.yml создал примеры сообщений о высокой нагрузке процессора и недоступности nginx:

groups:
  - name: example-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance)(irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100 > 80
        for: 5m
        labels:
          severity: "warning"
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}% for 5 minutes."

      - alert: ServiceDown
        expr: up{job="nginx"} == 0
        for: 1m
        labels:
          severity: "critical"
        annotations:
          summary: "Service {{ $labels.instance }} is DOWN!"
          description: "{{ $labels.job }} on {{ $labels.instance }} is not responding."

4. Составил конфигурационный файл alertmanager.yml:

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h
  receiver: 'default-receiver'
  routes:
    - match:
        severity: 'critical'
      receiver: 'telegram-critical'
    - match:
        severity: 'warning'
      receiver: 'email-warning'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: '<ЛОГИН>@yandex.ru'
        from: '<ЛОГИН>@yandex.ru'
        smarthost: 'smtp.yandex.ru:587'
        auth_username: '<ЛОГИН>@yandex.ru'
        auth_password: '<ПАРОЛЬ>'
        require_tls: true

  - name: 'telegram-critical'
    telegram_configs:
      - api_url: "https://api.telegram.org"
        bot_token: "<ТОКЕН БОТА>"
        chat_id: <ID КАНАЛА, КУДА НАДО ПРИСЛАТЬ СООБЩЕНИЯ>
        parse_mode: "HTML"
        message: "🔥 <b>CRITICAL ALERT</b> 🔥\n{{ .CommonAnnotations.summary }}\n{{ .CommonAnnotations.description }}"

  - name: 'email-warning'
    email_configs:
      - to: '<ЛОГИН>@yandex.ru'
        from: '<ЛОГИН>@yandex.ru'
        smarthost: 'smtp.yandex.ru:587'
        auth_username: '<ЛОГИН>@yandex.ru'
        auth_password: '<ПАРОЛЬ>'
        require_tls: true

