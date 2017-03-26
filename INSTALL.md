---
layout: page
title: Install
order: 3
---
# Установка

Инструкции по развёртыванию облачного сервиса Archery

## Эксплуатационный режим

Данная схема развёртывания предназначена для использования в боевом режиме. Используйте её, если вам нужна масштабируемость и производительность.

### Список компонент

1. **Contol Panel Instance** - Контрольная панель и главная страница сервиса
1. **Designer Instance** - Компонент, отвечающий за режим конструирования портала
1. **Viewer Instance** - Компонент, отвечающий за режим отображения портала пользователям
1. **Storage Instance** - Сервис доступа к данным структуры порталов
1. **Source Instance** - Демонстрационный сервис доступа к данным
1. **Nginx** - Прокси-сервер Nginx, отвечающий за маршрутизацию

### Принципиальная схема

**Принципиальная схема**
[(Full Size)](/public/install/Production.png)

![Production](/public/install/Production.png)

### Инструкции по установке

Перед началом установки нужно убедиться, что установлены следующие компоненты системы:

1. npm 3.10.10+
1. node v7.2.1+
1. gulp CLI version 1.2.2+
1. bower 1.7.9+
1. pm2 2.2.3+
1. jdk 1.8.0_102+
1. maven 3.16+
1. nginx 1.10.1+
1. mysql server 5.6.29+

Инструкции написаны для `Debian GNU/Linux 8 (jessie)`. Некоторые компоненты не вляются строго необходимыми и могут быть заменены, другие компоненты нужны только при сборке платформы и не нужны для её работы, однако инструкции составлены в предположении, что исопльзуются именно эти компоненты.

#### Настройка прокси-сервера Nginx

В конфигурационный каталог прокси-сервера Nginx нужно разместить следующий конфигурационный файл:

`/etc/nginx/sites-available/com.example.conf`
```
server {

    listen          *:80;
    server_name     archery.example.com;

    access_log      /var/log/nginx/com.example.archery.access.log;
    error_log       /var/log/nginx/com.example.archery.error.log;

    gzip            on;
    gzip_static     on;
    gzip_disable    "MSIE [1-6]\.(?!.*SV1)";

    location / {

        proxy_pass              http://127.0.0.1:3000;
        proxy_set_header        X-Forwarded-Host        $host;
        proxy_set_header        X-Forwarded-Server      $host;
        proxy_set_header        X-Forwarded-For         $proxy_add_x_forwarded_for;
    }
}

server {

    listen          *:80;
    server_name     storage.example.com;

    access_log      /var/log/nginx/com.example.storage.access.log;
    error_log       /var/log/nginx/com.example.storage.error.log;

    gzip            on;
    gzip_static     on;
    gzip_disable    "MSIE [1-6]\.(?!.*SV1)";

    location / {

        proxy_pass              http://127.0.0.1:8104;
        proxy_set_header        X-Forwarded-Host        $host;
        proxy_set_header        X-Forwarded-Server      $host;
        proxy_set_header        X-Forwarded-For         $proxy_add_x_forwarded_for;

        set    $auto_x_client_host    "";

        if ($http_referer ~* ^http(s?)://(localhost:3000|archery.example.com)/(edit|view)/([1-9][0-9]*)/(.*)$) {
            set    $auto_x_client_host    $4.si.example.com;
        }

        set    $if_x_client_host    $http_x_client_host;

        if ($if_x_client_host = "") {
            set    $if_x_client_host    $auto_x_client_host;
        }

        proxy_set_header        X-Client-Host         $if_x_client_host;
    }
}

server {

    listen          *:80;
    server_name     *.ui.example.com;

    access_log      /var/log/nginx/com.example.ui.access.log;
    error_log       /var/log/nginx/com.example.ui.error.log;

    gzip            on;
    gzip_static     on;
    gzip_disable    "MSIE [1-6]\.(?!.*SV1)";

    location / {

        proxy_pass              http://127.0.0.1:3001;
        proxy_set_header        X-Forwarded-Host        $host;
        proxy_set_header        X-Forwarded-Server      $host;
        proxy_set_header        X-Forwarded-For         $proxy_add_x_forwarded_for;
    }
}
```

Данный конфигурационный файл настраивает платформу на работу через поддомены на домене example.com. Для работы платформы на другом домене требуется заменить везде в тексте example.com (и com.example) на другое имя.

- Зона `*.ui.example.com` это зона, где конечные пользователи платформы смогут свободно создавать поддомены. Посредством этих доменов пользователи получают доступ к компоненту Viewer.
- Зона `*.si.example.com` это зона, где платформа будет создавать служебные поддомены, необходимые для работы компонента Designer.
- Домен `archery.example.com` это публичный домен, через который пользователи смогут работать с платформой (c компонентами Control Panel, Designer, Viewer).
- Домен `storage.example.com` это публичный домен, на котором будут размещены API компонент Storage и Source.

В Production-окружении нагрузка на все компоненты может быть сбалансирована посредством Nginx-инструкции `upstream`. Такая конфигурация уже специфична для установки.

Также нуобходимо создать символическую ссылку на данный файл конфигурации в конфигурационном каталоге прокси-севера Nginx `sites-available` и перезапустить сервер Nginx:

```
ln -s /etc/nginx/sites-available/com.example.conf /etc/nginx/sites-enabled/com.example.conf`
sevice nginx reload
```

#### Настройка клиентской части платформы

Далее следует настроить компоненты Control Panel. Designer, Viewer. В домашнем каталоге администратора системы следует разместить следующий исполняемый файл:

`~/scripts/checkout-com.example.archery`

```
#!/bin/bash

SERVICE='com.example.archery'
PROJECT_DIR="/var/projects/$SERVICE"

cd $PROJECT_DIR

git clone https://github.com/ntr1x/ntr1x-archery.git
git clone https://github.com/ntr1x/ntr1x-archery-core.git
git clone https://github.com/ntr1x/ntr1x-archery-shell.git
git clone https://github.com/ntr1x/ntr1x-archery-landing.git
git clone https://github.com/ntr1x/ntr1x-archery-widgets.git
git clone https://github.com/ntr1x/ntr1x-archery-widgets-academy.git

chmod -R 775 $PROJECT_DIR
```

Следует также вручную создать каталог:

`/var/projects/com.example.archery`

Вместо `com.example` в имени файла и тексте сценария следует исопльзовать домен, на поддоменах которого разворачивается платформа.

Далее следует выполнить этот сценарий:
```
sh ~/scripts/checkout-com.example.archery
```

Эта команда загрузить дистрибутив из Git-репозитария.

Далее следует собрать платформу из исходных кодов. Для этого в домашней директории администратора системы следует создать следующий файл:

`~/scripts/build-com.example.archery`

```
#!/bin/bash

SERVICE='com.example.archery'
PROJECT_DIR="/var/projects/$SERVICE"
WEBAPP_DIR="/opt/$SERVICE"

cd $PROJECT_DIR/ntr1x-archery-core
git reset --hard
git pull
npm i

cd $PROJECT_DIR/ntr1x-archery-landing
git reset --hard
git pull
npm i

cd $PROJECT_DIR/ntr1x-archery-shell
git reset --hard
git pull
npm i

cd $PROJECT_DIR/ntr1x-archery-widgets
git reset --hard
git pull
npm i

cd $PROJECT_DIR/ntr1x-archery-widgets-academy
git reset --hard
git pull
npm i

cd $PROJECT_DIR/ntr1x-archery
git reset --hard
git pull
npm i

chmod -R 775 $PROJECT_DIR

pm2 stop all

rm -rf $WEBAPP_DIR/*
mkdir -p $WEBAPP_DIR
chmod 775 $WEBAPP_DIR
cp -R $PROJECT_DIR/ntr1x-archery/. $WEBAPP_DIR
rm -rf $WEBAPP_DIR/.git
cd $WEBAPP_DIR
chgrp -R developers .

pm2 start ./www/bin/server
pm2 start ./www/bin/viewer

```

Следует также вручную создать каталог:

`/opt/com.example.archery`

Далее следует выполнить этот сценарий:
```
sh ~/scripts/build-com.example.archery
```

Эта команда обновит дистрибутив и перезапустит все компоненты. Компоненты Control Panel и Designer будут запущены на порте 3000. Компонент Viewer будет запущен на порте 3001.

#### Настройка серверной части платформы

## Режим разработчика

Данная схема развёртывания предназначена для использования в режиме разработчика.

### Список компонент

1. **Server Instance** - Контрольная панель и главная страница сервиса
1. **Backend Instance** - Компонент, отвечающий за режим конструирования портала

### Принципиальная схема

**Принципиальная схема**
[(Full Size)](/public/install/Dev.png)

![Production](/public/install/Dev.png)

### Инструкции по установке

TODO
