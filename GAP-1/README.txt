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
