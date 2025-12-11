---
layout: default
title: Developement Module
nav_order: 80
tags:
  - devops
  - linux
  - nginx
  - letsencrypt
  - php
  - nodejs
  - ruby
  - python
  - java
  - aspnet
  - elixir
  - django
  - uwsgi
---

## PHP Symfony
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


## NodeJS Deploy


---
Сборка проекта

```
useradd -s /bin/bash -d /home/node node
su - node

git clone https://github.com/nodejs/nodejs.org.git

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash

nvm install v14.17.5
nvm use v14.17.5

cd nodejs.org/

npm install

exit

```

Установка и настрока nginx
```
apt install -y certbot nginx 

systemctl enable nginx
systemctl stop nginx  # останавливаем чтобы получить сертификат через certbot в standalone режиме

certbot certonly -d nur76n-dev.devops.rebrain.srwx.net

rm /etc/nginx/sites-enabled/default  
```
Содержимое конфиг файла nginx /etc/nginx/sites-enabled/node
```
map $http_accept_language $index_redirect_uri {
	default "/en/";
	"~(^|,)en.+,ru" "/en/";
	"~(^|,)ru.+,en" "/ru/";
	"~(^|,)en" "/en/";
	"~(^|,)ru" "/ru/";
}

server {
	listen 80;

	server_name nur76n-dev.devops.rebrain.srwx.net;

	location / {
		return 301 https://$host$request_uri;
	}
}

server {
	listen 443 ssl;

	root /home/node/nodejs.org/build;

	index index.html;

	server_name nur76n-dev.devops.rebrain.srwx.net;

	ssl_certificate /etc/letsencrypt/live/nur76n-dev.devops.rebrain.srwx.net/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/nur76n-dev.devops.rebrain.srwx.net/privkey.pem; 

	location = / {
		return 302 $index_redirect_uri;
	}

	location / {
		try_files $uri $uri/ =404;
	}	
}

```
Запуск nginx
```
nginx -t
systemctl start nginx
```





## Ruby on Rails Deploy

Список команд:
```
installing postgres and dependencies
sudo apt-get install -y certbot nginx postgresql nodejs libv8-dev libpq5 libpq-dev

su - postgres
psql
create user ruby_user;
\password ruby_user;
create database ruby_db;
grant all privileges on database ruby_db to ruby_user;

\q

logout
Установка ruby
useradd -s /bin/bash -G sudo -m ruby
echo "ruby ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/92-ruby
chmod 0440 /etc/sudoers.d/92-ruby

su - ruby

gpg2 --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
\curl -sSL https://get.rvm.io | bash -s stable

logout 
su - ruby 

rvm install ruby-2.7.2

git clone https://github.com/1sherlynn/alphacamp_blog_app.git

cd alphacamp_blog_app

gem install bundler:1.15.4

bundler install

```


```
Добавляем в файл config/database.yml
production:
  <<: *default
  host: localhost
  adapter: postgresql
  encoding: utf8
  database: ruby_db
  pool: 5
  username: ruby_user
  password: password1234

RAILS_ENV=production rake db:migrate

RAILS_ENV=production rake assets:precompile

RAILS_ENV=production rake secret # сохраняем вывод и добавляем в файл systemd puma

mkdir config/puma
```
Содержимое файла /home/ruby/alphacamp_blog_app/config/puma/production.rb

```
rails_env = "production"
environment rails_env

app_dir = "/home/ruby/alphacamp_blog_app" # Update me with your root rails app path

bind  "unix://#{app_dir}/puma.sock"
pidfile "#{app_dir}/puma.pid"
state_path "#{app_dir}/puma.state"
directory "#{app_dir}/"

stdout_redirect "#{app_dir}/log/puma.stdout.log", "#{app_dir}/log/puma.stderr.log", true

workers 1
threads 1,2

activate_control_app "unix://#{app_dir}/pumactl.sock"

prune_bundler
logout
```

### Дальше под рутом

Содержимое файла /etc/systemd/system/puma.service
```
[Unit]
Description=Puma HTTP Server
After=network.target

[Service]
Type=simple
User=ruby
WorkingDirectory=/home/ruby/alphacamp_blog_app
Environment=RAILS_ENV=production
Environment=SECRET_KEY_BASE='5903f4f61848092eca35bdd0255d65eb0eef11b919f82ada12bef0a896e2e252fdf511db3b709b4703feaa82d82488ef3a7779b9283b8cb71662a2531845451e'


ExecStart=/home/ruby/.rvm/gems/ruby-2.7.2/wrappers/puma -C /home/ruby/alphacamp_blog_app/config/puma/production.rb
Restart=always
KillMode=process

[Install]
WantedBy=multi-user.target

```

```
systemctl daemon-reload
systemctl enable puma
systemctl start puma
systemctl status puma
Настройка nginx
systemctl stop nginx
rm /etc/nginx/sites-enabled/default

certbot certonly -n --standalone --agree-tos -m nur76n@mail.ru -d nur76n-ruby.devops.rebrain.srwx.net
```
Содержимое файла /etc/nginx/sites-enabled/ruby
```
upstream app {
  server unix:///home/ruby/alphacamp_blog_app/puma.sock  fail_timeout=0;
}

server {
  listen 80;
  server_name nur76n-ruby.devops.rebrain.srwx.net;

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl;
  server_name  nur76n-ruby.devops.rebrain.srwx.net;
  ssl_certificate /etc/letsencrypt/live/nur76n-ruby.devops.rebrain.srwx.net/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/nur76n-ruby.devops.rebrain.srwx.net/privkey.pem; 

  root /home/ruby/alphacamp_blog_app/public;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  location / {
    proxy_pass http://app;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Host 'nur76n-ruby.devops.rebrain.srwx.net';
    proxy_set_header X-Forwarded-Proto $scheme;
  }

  location ~ ^/(500|404|422).html {
    root /path/to/rails/public;
  }

  error_page 500 502 503 504 /500.html;
  error_page 404 /404.html;
  error_page 422 /422.html;

  client_max_body_size 4G;
  keepalive_timeout 10;
}
```

```
nginx -t
systemctl start nginx
```




## Python Deploy

Список команд:
```
useradd -s /bin/bash -G sudo -m python
echo "python ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/92-python
chmod 0440 /etc/sudoers.d/92-python

su - python

sudo apt-get install -y certbot nginx make build-essential \
  libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev \
  wget curl llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev \
  libffi-dev liblzma-dev postgresql libpq5 libpq-dev


sudo -u postgres psql
create user django_user;
\password django_user
create database django_db;
grant all privileges on database django_db to django_user;
\q

git clone https://github.com/pyenv/pyenv.git ~/.pyenv

cd ~/.pyenv && src/configure && make -C src

### Put these lines into ~/.profile before the part that sources ~/.bashrc:
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"

### And put this line at the bottom of ~/.profile:
eval "$(pyenv init --path)"

logout
su - python

git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc

logout
su - python

pyenv install 3.7.1
pyenv global 3.7.1

git clone https://github.com/gothinkster/django-realworld-example-app
cd django-realworld-example-app

pyenv virtualenv 3.7.1 app
pyenv local app

pip install -r requirements.txt 
pip install psycopg2==2.8.6
pip install uwsgi
```

Меняем параметры подключения к БД conduit/settings.py и значения переменных STATIC_ROOT, ALLOWED_HOSTS
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'django_db',
        'USER': 'django_user',
        'PASSWORD': 'password1234',
        'HOST': '127.0.0.1',
        'PORT': '5432',
    }
}

STATIC_ROOT = os.path.join(BASE_DIR, "static")

ALLOWED_HOSTS = ['nur76n-python.devops.rebrain.srwx.net']
python manage.py migrate

mkdir ./static

python manage.py collectstatic

python manage.py createsuperuser
```
Содержимое файла ~/django-realworld-example-app/django_uwsgi.ini
```
[uwsgi]
# full path to Django project's root directory
chdir            = /home/python/django-realworld-example-app/
# Django's wsgi file
module           = conduit.wsgi
# full path to python virtual env
home             = /home/python/.pyenv/versions/app
# enable uwsgi master process
master          = true
# maximum number of worker processes
processes       = 10
# the socket (use the full path to be safe
socket          = /home/python/django-realworld-example-app/django_uwsgi.sock
# socket permissions
chmod-socket    = 666
# clear environment on exit
vacuum          = true
# daemonize uwsgi and write messages into given log
# daemonize       = /home/python/django-realworld-example-app/uwsgi.log
```


Содержимое файла /etc/systemd/system/uwsgi@django.service
```
[Unit]
Description=uWSGI django
After=syslog.target

[Service]
ExecStart=/home/python/.pyenv/versions/app/bin/uwsgi --ini  /home/python/django-realworld-example-app/django_uwsgi.ini
# Requires systemd version 211 or newer
RuntimeDirectory=/home/python/django-realworld-example-app
Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable uwsgi@django.service
sudo systemctl start uwsgi@django.service
sudo systemctl status uwsgi@django.service
```


Содержимое файла ~/django-realworld-example-app/uwsgi_params
```
uwsgi_param  QUERY_STRING       $query_string;
uwsgi_param  REQUEST_METHOD     $request_method;
uwsgi_param  CONTENT_TYPE       $content_type;
uwsgi_param  CONTENT_LENGTH     $content_length;
uwsgi_param  REQUEST_URI        $request_uri;
uwsgi_param  PATH_INFO          $document_uri;
uwsgi_param  DOCUMENT_ROOT      $document_root;
uwsgi_param  SERVER_PROTOCOL    $server_protocol;
uwsgi_param  REQUEST_SCHEME     $scheme;
uwsgi_param  HTTPS              $https if_not_empty;
uwsgi_param  REMOTE_ADDR        $remote_addr;
uwsgi_param  REMOTE_PORT        $remote_port;
uwsgi_param  SERVER_PORT        $server_port;
uwsgi_param  SERVER_NAME        $server_name;
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl stop nginx
sudo certbot certonly -n --standalone --agree-tos -m nur76n@mail.ru -d nur76n-python.devops.rebrain.srwx.net
Содержимое файла /etc/nginx/sites-enabled/django

# the upstream component nginx needs to connect to
upstream django {
    server unix:///home/python/django-realworld-example-app/django_uwsgi.sock;
}
# configuration of the server

server {
    listen 80;
    server_name nur76n-python.devops.rebrain.srwx.net;

    location / {
      return 301 https://$host$request_uri;
  }
}

server {
    listen      443 ssl;
    server_name nur76n-python.devops.rebrain.srwx.net;
    charset     utf-8;
    
    ssl_certificate /etc/letsencrypt/live/nur76n-python.devops.rebrain.srwx.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nur76n-python.devops.rebrain.srwx.net/privkey.pem; 

    location /static {
        alias /home/python/django-realworld-example-app/static;
    }
    # Send all non-media requests to the Django server.
    location / {
        uwsgi_pass  django;
        include     /home/python/django-realworld-example-app/uwsgi_params;
    }
}
```

```
sudo nginx -t
sudo systemctl start nginx
```




## NodeJS Hubot Deploy

Список команд:
```
useradd -s /bin/bash -m nodejs
su - nodejs 

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
/bin/bash
nvm install v14.17.5
nvm use v14.17.5
npm install -g yo generator-hubot 

mkdir myhubot && cd $_
yo hubot

nano external-scripts.json ## Delete line hubot-redis and node-heroku-keepalive

export TELEGRAM_TOKEN=1927019038:AAGSKiAsVTscA2whi0J5xnCJI3hxAh1oEyk
```

nano scripts/test.coffee
```
module.exports = (robot) ->

  robot.hear /badger/i, (res) ->
    res.send "Badgers? BADGERS? WE DON'T NEED NO STINKIN BADGERS"

  robot.respond /multiply (.*) and (.*)/i, (res) ->
    val1 = res.match[1]
    val2 = res.match[2]

    if val1 >  50
      res.reply "Too hard"
    else
      res.reply "result is " + String(val1 * val2)

  robot.respond /time in (.*)/i, (res) ->
    today = new Date
    hour = today.getHours()
    minute = today.getMinutes()

    city = res.match[1]
    ctime = switch city
      when "Almaty","almaty","alm" then  hour + ":" + minute
      when "Moscow","moscow","msk" then (hour - 3) + ":" + minute
      when "Kiev","kiev" then (hour - 3) + ":" + minute
      when "London","london" then (hour - 5) + ":" + minute
      else "I don't know"
    res.reply "Time in " + city + " now " + ctime
bin/hubot -a telegram
```


nano /etc/systemd/system/hubot.service

```
[Unit]
Description=Hubot
Requires=network.target
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/nodejs/myhubot
User=nodejs

Restart=always
RestartSec=10

; Configure Hubot environment variables, use quotes around vars with whitespace as shown below.
Environment=TELEGRAM_TOKEN=1927019038:AAGSKiAsVTscA2whi0J5xnCJI3hxAh1oEyk
Environment=PATH="/home/nodejs/.nvm/versions/node/v14.17.5/bin"
; Alternatively multiple environment variables can loaded from an external file
;EnvironmentFile=/etc/hubot-environment

ExecStart=/home/nodejs/myhubot/node_modules/.bin/coffee node_modules/hubot/bin/hubot -a telegram

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload 
systemctl start hubot.service
systemctl status hubot.service
```



## Java Deploy

Список команд:

```
sudo apt-get install openjdk-8-jdk maven nginx certbot

useradd -s /bin/bash -m java
su - java


git clone https://github.com/otale/tale.git
cd tale
git checkout v2.0.5

mvn clean package -Pprod -Dmaven.test.skip=true

cd target/dist
mkdir ~/tale_app
tar -xvf tale.tar.gz -C ~/tale_app/
cd ~/tale_app/

```

В файле resources/application.properties изменить следующие параметры
```
app.devMode=false
com.blade.logger.org.sql2o=warn

```

Создаем файл /etc/systemd/system/java_tale_app.service
```
[Unit]
Description=Java Tale App
After=network.target

[Service]
Type=simple
User=java
WorkingDirectory=/home/java/tale_app

ExecStart=java -Xms256m -Xmx256m -Dfile.encoding=UTF-8 -jar tale-latest.jar --app.env=prod
Restart=always
KillMode=process

[Install]
WantedBy=multi-user.target

```

```
systemctl daemon-reload
systemctl enable java_tale_app
systemctl start java_tale_app
systemctl status java_tale_app

```

Добавляем правила и включаем firewall
```
ufw allow in ssh
ufw allow in http
ufw allow in https
ufw enable 
systemctl stop nginx
rm /etc/nginx/sites-enabled/default

certbot certonly -n --standalone --agree-tos -m nur76n@mail.ru -d nur76n-java.devops.rebrain.srwx.net
```

Cоздаем файл /etc/nginx/sites-enabled/java_tale_app
```
server {
  listen 80;
  server_name nur76n-java.devops.rebrain.srwx.net;

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl;
  server_name nur76n-java.devops.rebrain.srwx.net;
  access_log off;

  ssl_certificate /etc/letsencrypt/live/nur76n-java.devops.rebrain.srwx.net/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/nur76n-java.devops.rebrain.srwx.net/privkey.pem; 

  location / {
    proxy_pass http://127.0.0.1:9000;
    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;
  }
}

```

```
nginx -t
systemctl start nginx

```




## ASP.NET Deploy

Список команд:
```
apt install certbot nginx

wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt-get update;   sudo apt-get install -y apt-transport-https &&   sudo apt-get update &&   sudo apt-get install -y dotnet-sdk-5.0
sudo apt-get update;   sudo apt-get install -y apt-transport-https &&   sudo apt-get update &&   sudo apt-get install -y aspnetcore-runtime-5.0

sudo wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-2019.list)"
sudo apt install -y mssql-server
/opt/mssql/bin/mssql-conf setup

curl https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
curl https://packages.microsoft.com/config/ubuntu/20.04/prod.list | sudo tee /etc/apt/sources.list.d/msprod.list
sudo apt update 
sudo ACCEPT_EULA=Y apt install mssql-tools unixodbc-dev
useradd -s /bin/bash -m -G sudo aspnet
su - aspnet
git clone https://github.com/dotnet-architecture/eShopOnWeb.git
cd eShopOnWeb
dotnet tool install --global dotnet-ef
```


Меняем параметры подключения к БД в файле appsettings.json
```
  "ConnectionStrings": {
    "CatalogConnection": "Server=localhost;Database=eshop_data;User Id=sa; Password=Passwd1234;",
    "IdentityConnection": "Server=localhost;Database=eshop_identity;User Id=sa; Password=Passwd1234;"
  }

```


```
dotnet restore
dotnet tool restore
dotnet ef database update -c catalogcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
dotnet ef database update -c appidentitydbcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj

dotnet publish --configuration Release

logout
```
Создаем файл /etc/systemd/system/eshop_aspnet_app.service
```
[Unit]
Description=eShopOnWeb .NET Web App

[Service]
WorkingDirectory=/home/aspnet/eShopOnWeb/src/Web
ExecStart=/usr/bin/dotnet /home/aspnet/eShopOnWeb/src/Web/bin/Release/net5.0/Web.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=dotnet-eshop
User=aspnet
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```


```
systemctl daemon-reload
systemctl enable eshop_aspnet_app.service
systemctl start eshop_aspnet_app.service
systemctl status eshop_aspnet_app.service
systemctl stop nginx
rm /etc/nginx/sites-enabled/default
certbot certonly -n --standalone --agree-tos -m nur76n@mail.ru -d nur76n-aspnet.devops.rebrain.srwx.net

```


Создаем файл /etc/nginx/sites-enabled/aspnet
```
server {
  listen 80;
  server_name nur76n-aspnet.devops.rebrain.srwx.net;

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen        443 ssl;
  server_name   nur76n-aspnet.devops.rebrain.srwx.net;

  ssl_certificate /etc/letsencrypt/live/nur76n-aspnet.devops.rebrain.srwx.net/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/nur76n-aspnet.devops.rebrain.srwx.net/privkey.pem;


  location / {
    proxy_pass         https://127.0.0.1:5001;
    proxy_http_version 1.1;
    proxy_set_header   Upgrade $http_upgrade;
    proxy_set_header   Connection keep-alive;
    proxy_set_header   Host $host;
    proxy_cache_bypass $http_upgrade;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
  }
}

```


```
systemctl start nginx
```



## Elixir Deploy

Список команд:

```
sudo apt update
sudo apt install -y  postgresql nginx certbot gcc make gcc npm unzip

# for erlang
apt-get -y install build-essential autoconf m4 libncurses5-dev libwxgtk3.0-gtk3-dev libgl1-mesa-dev libglu1-mesa-dev libpng-dev libssh-dev unixodbc-dev xsltproc fop libxml2-utils libncurses-dev openjdk-11-jdk libssl-dev inotify-tools

sudo -u postgres psql
create user elixir;
alter user elixir createdb;
\password elixir
grant all privileges on database  cercle_db to elixir;
\q
useradd -s /bin/bash -m -G sudo elixir
su - elixir


git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.8.1
echo . $HOME/.asdf/asdf.sh >> ~/.bashrc
echo . $HOME/.asdf/completions/asdf.bash >> ~/.bashrc

logout
su - elixir

asdf plugin-add elixir
asdf install elixir 1.4
asdf global elixir 1.4


mkdir clones && cd clones


git clone https://github.com/openssl/openssl.git --branch OpenSSL_1_0_2-stable
cd openssl/
mkdir __result
./config --prefix="${HOME}/clones/openssl" shared zlib -fPIC
make depend
make
make install INSTALL_PREFIX="/home/${HOME}/clones/openssl/__result"

export KERL_CONFIGURE_OPTIONS="--enable-debug --without-javac --enable-shared-zlib --enable-dynamic-ssl-lib --enable-hipe --enable-sctp --enable-smp-support --enable-threads --enable-kernel-poll --enable-wx --with-ssl=${HOME}/clones/openssl/__result/${HOME}/clones/openssl/"

asdf plugin-add erlang
asdf install erlang 19.3.6
asdf global erlang 19.3.6

git clone https://github.com/cerclecrm/cercle.git
cd cercle

mv config/dev.secret_example.exs config/dev.secret.exs
```


Изменить параметры подключения к БД в файле dev.secret.exs
````
config :cercleApi, CercleApi.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "elixir",
  password: "password1234",
  database: "cercle_db",
  hostname: "localhost",
  pool_size: 10
```

```
mix deps.get
mix deps.update postgrex

# db migrations
mix ecto.create
mix ecto.migrate

npm install
npm install vue-select@2.5.0
```


nano /etc/systemd/system/cercle-crm.service
```
[Unit]
Description=cercle-crm

[Service]
Environment=PATH=/home/elixir/.asdf/shims:/home/elixir/.asdf/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Type=forking
WorkingDirectory=/home/elixir/cercle
ExecStart=/home/elixir/.asdf/shims/mix phoenix.server
Restart=always
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=cercle-crm
User=elixir

[Install]
WantedBy=multi-user.target

```

```
systemctl daemon-reload
systemctl enable cercle-crm.service
systemctl start cercle-crm.service
systemctl status cercle-crm.service
systemctl stop nginx
rm /etc/nginx/sites-enabled/default
certbot certonly -n --standalone --agree-tos -m nur76n@mail.ru -d nur76n-dev.devops.rebrain.srwx.net

```


Создаем файл /etc/nginx/sites-enabled/cercle
```
upstream phoenix_upstream {
  server 127.0.0.1:4000;
}

server {
  listen 80;
  server_name nur76n-dev.devops.rebrain.srwx.net;

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen        443 ssl;
  server_name   nur76n-dev.devops.rebrain.srwx.net;

  ssl_certificate /etc/letsencrypt/live/nur76n-dev.devops.rebrain.srwx.net/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/nur76n-dev.devops.rebrain.srwx.net/privkey.pem;
  location / {
    proxy_redirect off;
    proxy_pass http://phoenix_upstream;
  }
}

```

```
systemctl start nginx

ufw allow in ssh
ufw allow in http
ufw allow in https
ufw enable 

```



## mastodon Deploy

---
Список команд, которые были выполнены для достижения результата:
```
curl -sL https://deb.nodesource.com/setup_12.x | bash -


curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list

apt update

apt install -y \
  imagemagick ffmpeg libpq-dev libxml2-dev libxslt1-dev file git-core \
  g++ libprotobuf-dev protobuf-compiler pkg-config nodejs gcc autoconf \
  bison build-essential libssl-dev libyaml-dev libreadline6-dev \
  zlib1g-dev libncurses5-dev libffi-dev libgdbm-dev \
  nginx redis-server redis-tools postgresql postgresql-contrib \
  certbot yarn libidn11-dev libicu-dev libjemalloc-dev

adduser --disabled-login mastodon

su - mastodon

git clone https://github.com/rbenv/rbenv.git ~/.rbenv
cd ~/.rbenv && src/configure && make -C src
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
exec bash
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install 2.7.2
rbenv global 2.7.2

gem install bundler --no-document

exit
```


```
sudo fallocate -l 1G /swapfile
sudo dd if=/dev/zero of=/swapfile bs=1024 count=1048576
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

```
sudo -u postgres psql


CREATE USER mastodon CREATEDB;
\q


su - mastodon

git clone https://github.com/mastodon/mastodon.git live && cd live
git checkout $(git tag -l | grep -v 'rc[0-9]*$' | sort -V | tail -n 1)

bundle config deployment 'true'
bundle config without 'development test'
bundle install -j$(getconf _NPROCESSORS_ONLN)
yarn install --pure-lockfile

RAILS_ENV=production bundle exec rake mastodon:setup
# admin login nur76n@mail.ru pwd 2072ab1317011d0d26e753a0c742414b

exit

```

```
cp /home/mastodon/live/dist/nginx.conf /etc/nginx/sites-available/mastodon
ln -s /etc/nginx/sites-available/mastodon /etc/nginx/sites-enabled/mastodon
sed -i 's/example.com/nur76n-dev.devops.rebrain.srwx.net/g' /etc/nginx/sites-enabled/mastodon
## and uncomment ssl_certificate and ssl_certificate_key options

systemctl stop nginx

rm /etc/nginx/sites-enabled/default

certbot certonly -n --standalone --agree-tos -m nur76n@mail.ru -d nur76n-dev.devops.rebrain.srwx.net

systemctl start nginx

cp /home/mastodon/live/dist/mastodon-*.service /etc/systemd/system/

# nano /etc/systemd/system/mastodon-*.service

systemctl daemon-reload

systemctl enable --now mastodon-web mastodon-sidekiq mastodon-streaming

```