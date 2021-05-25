# ProcessMaker 4 Documentation

# Overview

ProcessMaker is an open source, workflow management software suite, which includes tools to automate your workflow, design forms, create documents, assign roles and users, create routing rules, and map an individual process quickly and easily. It's relatively lightweight and doesn't require any kind of installation on the client computer. This file describes the requirements and installation steps for the server.

## System Requirements

* [Ubuntu 20.04](https://releases.ubuntu.com/20.04/)
* [PHP 7.4](https://php.net)
* [PHP-FPM](https://www.php.net/manual/en/install.fpm.php)
* [PHP GD Extension](https://www.php.net/manual/en/image.installation.php)
* [PHP ImageMagick Extension](https://www.php.net/manual/en/book.imagick.php)
* [PHP IMAP Extension](https://www.php.net/manual/en/imap.setup.php)
* [Nginx](https://nginx.org/)
* [MySql 5.7](https://dev.mysql.com/downloads/mysql/5.7.html)
* [Redis](https://redis.io/)
* [Docker](https://docs.docker.com/get-docker/)
* [Composer 2](https://getcomposer.org/)
* [Node.js 14.4.0](https://nodejs.org/en/)
* [NPM 6.14](https://www.npmjs.com/package/npm)


## Install Required System Requirements
### Step 1. Install Ubuntu 20.04 LTS
[How to install Ubuntu 20.04 LTS](https://releases.ubuntu.com/20.04/)

### Step 2. Install Required Dependency (PHP, NGINX)

```
$ sudo apt-get update -y
$ sudo apt install -y \
		php-fpm php-cli php-json php-common php-mysql \
		php-zip php-gd php-mbstring php-curl php-xml \
		php-pear php-bcmath php-imagick php-dom php-sqlite3 \
		nginx vim curl unzip wget supervisor cron build-essential mysql-client
```

### Step 3. Install Docker
* Follow how to install Docker via [link](https://docs.docker.com/engine/install/ubuntu/)
* If you are behind procy, you need to config docker proxy follow [link](https://docs.docker.com/network/proxy/#use-environment-variables)

### Step 4. Run MySQL and Redis server
Create docker-compose and database env file:
```
cat <<EOF> db.env
MYSQL_ROOT_PASSWORD=password
MYSQL_PASSWORD=password
MYSQL_USER=processmaker
MYSQL_DATABASE=processmaker
EOF

cat << EOF > docker-compose.yml
version: "3.7"
services:
  redis:
    image: redis
    container_name: redis
    ports:
      - 6379:6379
    restart: always
  mysql:
    image: mariadb
    container_name: mariadb
    restart: always
    ports:
      - 3306:3306
    env_file:
      - db.env
    volumes:
      - database:/var/lib/mysql
  phpmyadmin:
    image: phpmyadmin
    container_name: phpmyadmin
    ports:
      - 81:80
    restart: always
    environment:
      PMA_HOST: mariadb
volumes:
  database:
EOF
```

Run docker-compose file:

```
sudo docker-compose up -d
```

### Step 5. Install NodeJs

```
$ curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
$ sudo apt -y install nodejs
```

### Step 6. Install Composer

```
$ wget -O composer-setup.php https://getcomposer.org/installer
$ sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
$ composer self-update
$ rm -rf composer-setup.php
```

### Step 7. Get config file

```
$ git clone https://github.com/ProcessMaker/pm4core-docker.git /tmp/pm4core-docker
```

### Step 8. Setting Cron

```
$ sudo cp -r /tmp/pm4core-docker/build-files/laravel-cron /etc/cron.d/laravel-cron
$ sudo chmod 0644 /etc/cron.d/laravel-cron && crontab /etc/cron.d/laravel-cron
```

### Step 9. Setup Supervisord

```
sudo bash -c 'cat << "EOF" > /etc/supervisor/conf.d/services.conf
[program:horizon]
directory=/code/pm4
command=php artisan horizon

[program:laravel-echo-server]
directory=/code/pm4
command=npx laravel-echo-server start

[program:cron]
command=cron -f
EOF'
```

### Step 10. Setup ProcessMaker

```
$ sudo mkdir -p /code
$ sudo chown $USER:$USER /code
$ sudo rm -rf /code/pm4
$ git clone https://github.com/BPMS-VTS/bpms.git /code/pm4

$ sudo chown $USER:$USER /code/pm4
$ cd /code/pm4
$ composer install

$ cp -r /tmp/pm4core-docker/build-files/laravel-echo-server.json /code/pm4
$ cp -r /tmp/pm4core-docker/build-files/init.sh /code/pm4
```

If you are behind proxy, do following:
```
$ git checkout /code/pm4/ProcessMaker/Console/Commands/BuildScriptExecutors.php
$ sed -i 's/docker build/docker build \-\-build\-arg http\_proxy\=http\:\/\/10\.61\.11\.42\:3128 \-\-build\-arg https\_proxy\=http\:\/\/10\.61\.11\.42\:3128/g' /code/pm4/ProcessMaker/Console/Commands/BuildScriptExecutors.php
```

Change database config in /code/pm4/init.sh

```
$ export PM_VERSION=4.1.0
$ export PM_APP_URL=http://localhost
$ export PM_APP_PORT=8080
$ export PM_BROADCASTER_PORT=6001
$ export PM_DOCKER_SOCK=/var/run/docker.sock
$ sed -i 's/h mysql/h 127\.0\.0\.1/g' /code/pm4/init.sh
$ sed -i 's/u pm/u processmaker/g' /code/pm4/init.sh
$ sed -i 's/ppass/ppassword/g' /code/pm4/init.sh
$ sed -i 's/host=mysql/host=127\.0\.0\.1/g' /code/pm4/init.sh
$ sed -i 's/host=redis/host=127\.0\.0\.1/g' /code/pm4/init.sh
$ sed -i 's/username=pm/username=processmaker/g' /code/pm4/init.sh
$ sed -i 's/password=pass/password=password/g' /code/pm4/init.sh
$ chmod +x /code/pm4/init.sh
```

Grant permission to processmaker source

```
$ sudo chown -R :www-data /code/pm4/storage/
$ sudo chown -R :www-data /code/pm4/bootstrap/cache/
$ sudo chmod -R 0777 /code/pm4/storage/
$ sudo chmod -R o+w /code/pm4/storage/
$ sudo chmod -R 0775 /code/pm4/bootstrap/cache/
$ sudo php artisan key:generate
```

Install processmaker

```
$ rm -rf .env
$ /code/pm4/init.sh
$ npm install --unsafe-perm=true && npm run dev
```

### Step 11. Setup Nginx

```
$ sudo rm -rf /etc/nginx/sites-available/processmaker.conf
$ sudo bash -c 'cat << "EOF" > /etc/nginx/sites-available/processmaker.conf
server {
    listen 80;
    server_name processmaker.local;
    root /code/pm4/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";

    index index.html index.htm index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
EOF'
$ sudo ln -s /etc/nginx/sites-available/processmaker.conf /etc/nginx/sites-enabled/
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo nginx -t
$ sudo systemctl enable nginx
$ sudo systemctl start nginx
$ sudo systemctl reload nginx
$ sudo systemctl restart nginx
```

### Step 12: For Debug
When developing, make sure to turn on debugging in your .env so you can see the actual error instead of the Whoops page.

```
APP_DEBUG=TRUE
```
