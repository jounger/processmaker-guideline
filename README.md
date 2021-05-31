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

### Step 2. Install Required Dependencies (NGINX, PHP and PHP Extensions)
```
$ sudo apt-get update -y
$ sudo apt install -y \
	php-fpm php-cli php-json php-common php-mysql \
	php-zip php-gd php-mbstring php-curl php-xml \
	php-pear php-bcmath php-imagick php-dom php-sqlite3 \
	nginx vim curl unzip wget supervisor cron build-essential mysql-client
```

### Step 3. Install Composer
```
$ cd /tmp
$ wget -O composer-setup.php https://getcomposer.org/installer
$ sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
$ composer self-update
```

### Step 4. Install NodeJS
```
$ curl -fsSL https://deb.nodesource.com/setup_14.x | sudo -E bash -
$ sudo apt-get install -y nodejs
```
Click [this link](https://github.com/nodesource/distributions#manual-installation) if you want to install other NodeJS version.

### Step 5. Install Docker
* [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
* [Install Docker Compose on Ubuntu](https://docs.docker.com/compose/install/)
* If you are behind proxy, check it out [this link](https://docs.docker.com/network/proxy/#use-environment-variables)

### Step 6. Setup MySQL and Redis server
Create a folder:
```
$ sudo mkdir -p /code/docker
$ cd /code/docker
```

Inside this folder create 2 file `db.env` and `docker-compose.yml`:
Create `db.env` file:
```
$ cat <<EOF> db.env
MYSQL_ROOT_PASSWORD=pass
MYSQL_PASSWORD=pass
MYSQL_USER=pm
MYSQL_DATABASE=processmaker
EOF
```

Create `docker-compose.yml` file:
```
$ cat << EOF > docker-compose.yml
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
    container_name: mysql
    restart: always
    ports:
      - 3306:3306
    env_file:
      - db.env
    volumes:
      - mysql-data:/var/lib/mysql
  phpmyadmin:
    image: phpmyadmin
    container_name: phpmyadmin
    ports:
      - 8080:80
    restart: always
    environment:
      PMA_HOST: mysql
volumes:
  mysql-data:
EOF
```

Run docker-compose:
```
$ sudo docker-compose up -d
```

### Step 7. Get PM4Core configuration files
```
$ git clone https://github.com/ProcessMaker/pm4core-docker.git /tmp/pm4core-docker
```

### Step 8. Setting Cron Jobs
```
$ sudo cp -r /tmp/pm4core-docker/build-files/laravel-cron /etc/cron.d/laravel-cron
$ sudo chmod 0644 /etc/cron.d/laravel-cron && crontab /etc/cron.d/laravel-cron
```

### Step 9. Setup Supervisord
```
$ sudo bash -c 'cat << "EOF" > /etc/supervisor/conf.d/services.conf
[program:horizon]
directory=/code/pm4
command=php artisan horizon
autostart=true
autorestart=true
user=root
numprocs=1
redirect_stderr=true

[program:laravel-echo-server]
directory=/code/pm4
command=npx laravel-echo-server start
autostart=true
autorestart=true
user=root
numprocs=1
redirect_stderr=true

[program:cron]
command=cron -f
EOF'
```

Useful commands:
```
$ sudo systemctl status supervisor
$ sudo systemctl start supervisor
$ sudo systemctl restart supervisor
```

If you have any issue with laravel-echo-server, you can use [pusher](https://pusher.com/). Create a channel with name `processmaker` and change value in `.env` file:
```
BROADCAST_DRIVER=pusher
...
PUSHER_APP_ID=your_app_id
PUSHER_APP_KEY=your_app_key
PUSHER_APP_SECRET=your_app_secret
PUSHER_CLUSTER=your_app_cluster
PUSHER_TLS=TRUE
PUSHER_DEBUG=FALSE
```

### Step 10. Setup ProcessMaker
1. Get Processmaker source:
```
$ sudo mkdir -p /code/pm4
$ git clone https://github.com/jounger/processmaker.git /code/pm4
```

2. Copy configuration files:
```
$ cp -r /tmp/pm4core-docker/build-files/laravel-echo-server.json /code/pm4
$ cp -r /tmp/pm4core-docker/build-files/init.sh /code/pm4
```

3. Change database configurations:
Export environment variables:
```
$ export PM_VERSION=4.1.18
$ export PM_APP_URL=http://localhost
$ export PM_APP_PORT=8080
$ export PM_BROADCASTER_PORT=6001
$ export PM_DOCKER_SOCK=/var/run/docker.sock
```

Change name of containers by local IP address (127.0.0.1):
```
$ sed -i 's/h mysql/h 127\.0\.0\.1/g' /code/pm4/init.sh
$ sed -i 's/host=mysql/host=127\.0\.0\.1/g' /code/pm4/init.sh
$ sed -i 's/host=redis/host=127\.0\.0\.1/g' /code/pm4/init.sh
$ chmod +x /code/pm4/init.sh
```

If you are behind proxy, run commands bellow:
```
$ git checkout /code/pm4/ProcessMaker/Console/Commands/BuildScriptExecutors.php
$ sed -i 's/docker build/docker build \-\-build\-arg http\_proxy\=http\:\/\/0\.0\.0\.0\:0 \-\-build\-arg https\_proxy\=http\:\/\/0\.0\.0\.0\:0/g' /code/pm4/ProcessMaker/Console/Commands/BuildScriptExecutors.php
```
You need to replace `http\:\/\/0\.0\.0\.0\:0` by a real proxy. For example: `http\:\/\/10\.10\.10\.10\:8000`.

4. Grant permission to processmaker source
```
$ sudo chown -R :www-data /code/pm4/storage/
$ sudo chown -R :www-data /code/pm4/bootstrap/cache/
$ sudo chmod -R 0777 /code/pm4/storage/
$ sudo chmod -R o+w /code/pm4/storage/
$ sudo chmod -R 0775 /code/pm4/bootstrap/cache/
```
5. Install composer and node dependencies:
Composer:
```
$ cd /code/pm4
$ composer install
$ sudo php artisan key:generate
$ /code/pm4/init.sh
```

NPM:
```
$ cd /code/pm4
$ npm install --unsafe-perm=true
$ npm run dev
```

### Step 11. Setup Nginx
Create nginx configuration file for processmaker:
```
$ sudo bash -c 'cat << "EOF" > /etc/nginx/sites-available/processmaker.conf
server {
    listen 80;
    server_name localhost;
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
```
Then...
```
$ sudo ln -s /etc/nginx/sites-available/processmaker.conf /etc/nginx/sites-enabled/
$ sudo rm /etc/nginx/sites-enabled/default
$ sudo nginx -t
$ sudo systemctl enable nginx
$ sudo systemctl start nginx
$ sudo systemctl reload nginx
$ sudo systemctl restart nginx
```

### Notice: Debugging
Turn on debug mode in `.env` to trace detail errors instead of the Whoops page.
```
APP_DEBUG=TRUE
```

Enter http://localhost and use default account to login site:
- username: admin
- password: admin123

In case you setup Processmaker inside VirtualBox and want to export it outside, firewall must be install and allow what ports you want:
```
$ sudo apt-get install ufw
$ sudo ufw enable
$ sudo ufw allow ssh // ssh (22) to allow access to inside VM
$ sudo ufw allow 80 // 80 is application port
$ sudo ufw allow 6001 // 6001 is websocket port
$ sudo ufw status verbose
```

In VirtualBox forward above ports.

End.
