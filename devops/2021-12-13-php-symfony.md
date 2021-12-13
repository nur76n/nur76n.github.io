---
layout: default
title: PHP Symfony Deploy
nav_order: 80
---

# PHP Symfony Deploy

# Установка
---
Установка необходимых пакетов

```bash
apt install -y \
  certbot \
  mysql-server \
  nginx \
	php-dom \
	php-fpm  \
	php-mbstring \
	php-mysql \
	php-sqlite3 \
	unzip

```

Установка композера
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '756890a4488ce9024fc62c56153228907f1545c228516cbf63f885e036d37e9a59d27d63f46af1d4d07ee0f76181c7d3') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"

mv composer.phar /usr/local/bin/composer

```

Настройка MySQL
```
mysql

CREATE USER 'symfony'@'%' IDENTIFIED BY 'password1234';
CREATE DATABASE symfony;

GRANT ALL PRIVILEGES ON symfony.* TO 'symfony'@'%';

exit

```

Создание пользователя и настройка фреймворка
```
useradd -d /home/symfony -m -s/bin/bash symfony

su - symfony

git clone https://github.com/symfony/demo.git

cd demo

composer install

nano ./env
# Comment this line 
# DATABASE_URL=sqlite:///%kernel.project_dir%/data/database.sqlite
# Uncomment or add this line 
# DATABASE_URL="mysql://symfony:password1234@127.0.0.1:3306/symfony?serverVersion=5.7",

./bin/console doctrin:schema:create

./bin/console doctrine:fixtures:load

exit
```
Настройка nginx-fpm и ssl
```
rm /etc/nginx/sites-enabled/default

systemctl stop nginx

certbot -d nur76n-dev2.devops.rebrain.srwx.net

mkdir /var/www/letsencrypt

```

```
cat <<EOF > /etc/nginx/sites-enabled/symfony
server {
    listen 80;
    server_name nur76n-dev2.devops.rebrain.srwx.net;

    location / {
        return 301 https://\$host\$request_uri;
    }
		
}

server {
        listen 443 ssl;

        root /home/symfony/demo/public;

        index index.php;

        server_name nur76n-dev2.devops.rebrain.srwx.net;
				ssl_certificate /etc/letsencrypt/live/nur76n-dev2.devops.rebrain.srwx.net/fullchain.pem;
				ssl_certificate_key /etc/letsencrypt/live/nur76n-dev2.devops.rebrain.srwx.net/privkey.pem; 
				
        location / {
                try_files \$uri /index.php\$is_args\$args;
        }
				
				location ^~ /.well-known/acme-challenge/ {
            default_type "text/plain";
            root /var/www/letsencrypt;
        }

        location ~ ^/index\.php(/|\$) {
                fastcgi_pass unix:/run/php/php7.4-fpm.sock;     
                fastcgi_split_path_info ^(.+\.php)(/.*)\$;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME \$realpath_root\$fastcgi_script_name;
                fastcgi_param DOCUMENT_ROOT \$realpath_root;     
                internal;
        }       
}

EOF

```

```
nginx -t
systemctl restart nginx
```