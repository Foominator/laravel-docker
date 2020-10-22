# Docker for laravel

## PHP

<b>/docker-compose.yml</b>  
```
#PHP Service
  php:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: php
    volumes:
      - ./:/var/www/html
    ports:
      - "9000:9000"
    networks:
      - laravel
```

<b>/docker/Dockerfile</b>  
```
#PHP Service
FROM php:7.3-fpm

RUN docker-php-ext-install pdo pdo_mysql
```

## Nginx
<b>/docker-compose.yml</b>  
```
  #Nginx Service
  nginx:
    image: nginx:stable-alpine
    container_name: nginx
    depends_on:
      - php
      - mysql
    restart: unless-stopped
    tty: true
    volumes:
      - ./:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
      - "443:443"
    networks:
      - laravel
```

<b>/nginx/default.conf</b>  
```
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/html/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri = 404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

## Mysql
<b>/docker-compose.yml</b>  
```
  #MYSQL Service
  mysql:
    image: mysql:latest
    container_name: mysql
    restart: always
    tty: true
    ports:
      - "4306:3306"
    volumes:
      - ./docker/mysql:/var/lib/mysql
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      - MYSQL_DATABASE=${DB_DATABASE}
      - MYSQL_USER=${DB_USERNAME}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    networks:
      - laravel
```

<b>/.env</b>  
```
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```
#### After mysql container built and started
- Connect to DB <b>by port :4306</b> by your mysql client
- Create and configure mysql user
```
CREATE USER 'laravel_user'@'%' IDENTIFIED BY 'password'; # Create mysql user
GRANT ALL ON laravel.* TO 'laravel_user'@'%'; # Give permissions to new user
FLUSH PRIVILEGES;
```
- Change <b>.env</b> db configurations
<b>/.env</b>  
```
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel_user # change to created user name(previous step)
DB_PASSWORD=password # change to created user password(previous step)
```

## Redis
<b>/docker-compose.yml</b>  
```
  #REDIS Service
  redis:
    container_name: redis
    image: redis:6
    ports:
      - 16379:6379
    networks:
      - laravel
```
- composer require predis/predis
- Change redis host to container name in <b>.env</b> file
```
REDIS_HOST=redis 
REDIS_PASSWORD=null
REDIS_PORT=6379
```

## Crontab
#### For laravel schedule
<b>/docker/Dockerfile</b>  
```
RUN apt-get update && apt-get install -y cron

# Add crontab file in the cron directory
ADD docker/crontab /etc/cron.d/cron

# Give execution rights on the cron job
RUN chmod 0644 /etc/cron.d/cron

# Create the log file to be able to run tail
RUN touch /var/log/cron.log

# Run the command on container startup
CMD printenv > /etc/environment && echo "cron starting..." && (cron) && : > /var/log/cron.log && tail -f /var/log/cron.log
```
<b>/docker/crontab</b>  
```
* * * * * root /usr/local/bin/php -q -f /var/www/html/artisan schedule:run --no-ansi >> /var/log/cron.log 2>&1
# Don’t remove the empty line at the end of this file. It is required to run the cron job
```
