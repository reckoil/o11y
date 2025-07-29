ДЗ #6:
1. Установил Ubuntu Server 24.04, Docker, Docker Compose

2. Составил docker-compose.yml-файл следующего содержания:

services:
  wordpress:
    image: wordpress:php8.3-fpm
    container_name: wordpress
    volumes:
      - ./wordpress_data:/var/www/html
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
      - ./db_data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wpdb
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./wordpress_data:/var/www/html
    depends_on:
      - wordpress

  influxdb:
    image: influxdb:1.8
    container_name: influxdb
    ports:
      - "8086:8086"
    volumes:
      - ./influx_data:/var/lib/influxdb
      - ./influxdb.conf:/etc/influxdb/influxdb.conf
    environment:
      - INFLUXDB_DB=telegraf
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=adminpass
      - INFLUXDB_USER=telegraf
      - INFLUXDB_USER_PASSWORD=telegrafpass

  chronograf:
    image: chronograf:1.8
    container_name: chronograf
    depends_on:
      - influxdb
    ports:
      - "8888:8888"
    volumes:
      - ./chronograf_data:/var/lib/chronograf
      - ./chronograf.conf:/etc/chronograf/chronograf.conf
    environment:
      - INFLUXDB_URL=http://192.168.64.136:8086
      - INFLUXDB_USERNAME=telegraf
      - INFLUXDB_PASSWORD=telegrafpass

  kapacitor:
    image: kapacitor:1.6
    container_name: kapacitor
    depends_on:
      - influxdb
    ports:
      - "9092:9092"
    volumes:
      - ./kapacitor_data:/var/lib/kapacitor
      - ./kapacitor.conf:/etc/kapacitor/kapacitor.conf
    environment:
      - KAPACITOR_INFLUXDB_0_URLS_0=http://192.168.64.136:8086
      - KAPACITOR_INFLUXDB_0_USERNAME=admin
      - KAPACITOR_INFLUXDB_0_PASSWORD=adminpass

  telegraf:
    image: telegraf:1.25.3
    container_name: telegraf
    depends_on:
      - influxdb
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf
      - /var/run/docker.sock:/var/run/docker.sock
    network_mode: "host"
    extra_hosts:
      - "wordpress:192.168.64.136"
      - "nginx:192.168.64.136"
    user: "0:0"
    environment:
      HOSTNAME: cms-monitoring

volumes:
  db_data: {}
  wordpress_data: {}
  influx_data: {}
  kapacitor_data: {}
  chronograf_data: {}

3. Создал файлы конфигураций Telegraf, InfluxDB, Kapacitor, Chronograf

4. Создал файл alerts.tick с правилами алертинга:

// Правило для высокой загрузки CPU
stream
    |from()
        .measurement('cpu')
        .where(lambda: "cpu" == 'cpu-total')
    |alert()
        .crit(lambda: "usage_user" > 90)
        .warn(lambda: "usage_user" > 70)
        .message('High CPU usage on {{ .Host }}: {{ index .Fields "usage_user" }}%')
        .log('/var/log/kapacitor/alerts.log')
        .email('admin@example.com')

// Правило для нехватки памяти
stream
    |from()
        .measurement('mem')
    |alert()
        .crit(lambda: "used_percent" > 90)
        .warn(lambda: "used_percent" > 80)
        .message('High memory usage on {{ .Host }}: {{ index .Fields "used_percent" }}%')
        .log('/var/log/kapacitor/alerts.log')

// Правило для 500 ошибок в Nginx
stream
    |from()
        .measurement('nginx')
    |alert()
        .crit(lambda: "server_errors" > 0)
        .message('Nginx 5xx errors detected on {{ .Host }}: {{ index .Fields "server_errors" }}')
        .log('/var/log/kapacitor/alerts.log')

// Правило для проверки доступности WordPress
stream
    |from()
        .measurement('http_response')
        .where(lambda: "component" == 'wordpress')
    |alert()
        .crit(lambda: "result_code" != 200)
        .message('WordPress unavailable! Status code: {{ index .Fields "result_code" }}')
        .log('/var/log/kapacitor/alerts.log')

// Правило для проверки доступности MySQL
stream
    |from()
        .measurement('mysql_status')
    |deadman(0.0, 1m)
        .message('MySQL service is down on {{ .Host }}')
        .log('/var/log/kapacitor/alerts.log')

5. Настроил дашборд, который в себе содержит следующие панели:
1) Активность процессора, памяти, I/O диска, сети, занятое дисковое пространство
2) Активность работы nginx, MySQL (количество соединений, время отклика, количество запросов) 

6. Настроил алертинг с отправкой на почту по чрезмерному потреблению ресурсов ВМ и падению CMS.
