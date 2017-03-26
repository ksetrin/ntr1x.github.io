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

Разместите в домашней папке администратора системы файл с настройками:

`~/scripts/config-com.example.storage/application-prod.properties`

```
spring.datasource.url = jdbc:mysql://localhost:3306/storage_dist?useSSL=false&autoReconnect=true&allowMultiQueries=true&useUnicode=true&characterEncoding=UTF-8&characterSetResults=utf8&connectionCollation=utf8_general_ci

spring.datasource.username = storage
spring.datasource.password = storage

eclipselink.ddl-generation.output-mode = sql-script
spring.jpa.show-sql = true

server.port = 8104

app.public.url = http://storage.example.com
app.public.host = storage.example.com
app.public.schemes = http

app.local.url = http://localhost:8104
app.local.host = localhost:8104

app.files.root = /var/data/com.example.storage
```

Создайте на сервере mysql базу данных с именем storage_dist, а также пользователя storage
с паролем storage с привилегиями на все операции над базой storage_dist. Или используйте
любые другие учётные данные, только убедитесь, что в конфигурационном файле указаны именно они.

Создайте вручную каталоги:

`/var/data/com.example.storage`
`/var/projects/com.example.storage`
`/opt/com.example.storage`

Разместите с домашней папке администратора системы сценарий загрузки и установки:

`~/scripts/build-com.example.storage`

```
#!/bin/bash

SERVICE='com.example.storage'
PROJECT_DIR="/var/projects/$SERVICE"
WEBAPP_DIR="/opt/$SERVICE"
GIT_PATH='https://github.com/ntr1x/ntr1x-storage-launch.git'
GIT_BRANCH='master'
MAVEN='/usr/bin/mvn'
MAVEN_LOCAL='~/.m2/repository'
CONFIG_FILE="~/scripts/config-$SERVICE/application-prod.properties"

rm -rf $PROJECT_DIR
mkdir -p $PROJECT_DIR
chmod 775 $PROJECT_DIR
git clone -b $GIT_BRANCH $GIT_PATH $PROJECT_DIR
rm -rf $PROJECT_DIR/.git
cd $PROJECT_DIR
chgrp -R developers .

#screen -S $SERVICE -X quit

rm -rf $WEBAPP_DIR/*
mkdir -p $WEBAPP_DIR
chmod 775 $WEBAPP_DIR
cp -R $PROJECT_DIR/. $WEBAPP_DIR
cp $CONFIG_FILE $WEBAPP_DIR/src/main/resources/
cd $WEBAPP_DIR
chgrp -R developers .
rm -rf $MAVEN_LOCAL/com/ntr1x/storage

#screen -S $SERVICE -d -m $MAVEN spring-boot:run -Drun.profiles=prod -P build
$MAVEN spring-boot:run -Drun.profiles=prod,recreate,setup -P build
#$MAVEN spring-boot:run -Drun.profiles=prod -P build
```

Обратите внимание на наличие закоментированных строк. Так и должно быть. Выполните этот сценарий командой:

```
sh ~/scripts/build-com.example.storage
```

Дождитесь запуска сервера и остановите его сочетанием CTRL+C.

Сценарий создаст в базе данных необходимые таблицы и наполнит их данными.
Сценарий не создаст некоторые необходимые индексы и их нужно создать в
консоли mysql-сервера, выполнив набор команд для базы данных `storage_dist`:

```
ALTER TABLE `portals` DROP FOREIGN KEY `FK_portals_UserId`;
ALTER TABLE `portals` ADD CONSTRAINT `FK_portals_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `portals` DROP FOREIGN KEY `FK_portals_ThumbnailId`;
ALTER TABLE `portals` ADD CONSTRAINT `FK_portals_ThumbnailId` FOREIGN KEY (`ThumbnailId`) REFERENCES `resources` (`Id`) ON DELETE SET NULL;

ALTER TABLE `domains` DROP FOREIGN KEY `FK_domains_PortalId`;
ALTER TABLE `domains` ADD CONSTRAINT `FK_domains_PortalId` FOREIGN KEY (`PortalId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `templates` DROP FOREIGN KEY `FK_templates_PortalId`;
ALTER TABLE `templates` ADD CONSTRAINT `FK_templates_PortalId` FOREIGN KEY (`PortalId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `grants` DROP FOREIGN KEY `FK_grants_UserId`;
ALTER TABLE `grants` ADD CONSTRAINT `FK_grants_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `offers` DROP FOREIGN KEY `FK_offers_UserId`;
ALTER TABLE `offers` ADD CONSTRAINT `FK_offers_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `offers` DROP FOREIGN KEY `FK_offers_RelateId`;
ALTER TABLE `offers` ADD CONSTRAINT `FK_offers_RelateId` FOREIGN KEY (`RelateId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `offers` DROP FOREIGN KEY `FK_offers_ImageId`;
ALTER TABLE `offers` ADD CONSTRAINT `FK_offers_ImageId` FOREIGN KEY (`ImageId`) REFERENCES `resources` (`Id`) ON DELETE SET NULL;

ALTER TABLE `orders` DROP FOREIGN KEY `FK_orders_UserId`;
ALTER TABLE `orders` ADD CONSTRAINT `FK_orders_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `orders` DROP FOREIGN KEY `FK_orders_RelateId`;
ALTER TABLE `orders` ADD CONSTRAINT `FK_orders_RelateId` FOREIGN KEY (`RelateId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `params` DROP FOREIGN KEY `FK_params_RelateId`;
ALTER TABLE `params` ADD CONSTRAINT `FK_params_RelateId` FOREIGN KEY (`RelateId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `publications` DROP FOREIGN KEY `FK_publications_UserId`;
ALTER TABLE `publications` ADD CONSTRAINT `FK_publications_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `publications` DROP FOREIGN KEY `FK_publications_RelateId`;
ALTER TABLE `publications` ADD CONSTRAINT `FK_publications_RelateId` FOREIGN KEY (`RelateId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `publications` DROP FOREIGN KEY `FK_publications_ImageId`;
ALTER TABLE `publications` ADD CONSTRAINT `FK_publications_ImageId` FOREIGN KEY (`ImageId`) REFERENCES `resources` (`Id`) ON DELETE SET NULL;

ALTER TABLE `resources_images` DROP FOREIGN KEY `FK_resources_images_RelateId`;
ALTER TABLE `resources_images` ADD CONSTRAINT `FK_resources_images_RelateId` FOREIGN KEY (`RelateId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `resources_images` DROP FOREIGN KEY `FK_resources_images_ImageId`;
ALTER TABLE `resources_images` ADD CONSTRAINT `FK_resources_images_ImageId` FOREIGN KEY (`ImageId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `sessions` DROP FOREIGN KEY `FK_sessions_UserId`;
ALTER TABLE `sessions` ADD CONSTRAINT `FK_sessions_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `tokens` DROP FOREIGN KEY `FK_tokens_UserId`;
ALTER TABLE `tokens` ADD CONSTRAINT `FK_tokens_UserId` FOREIGN KEY (`UserId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `resources_uploads` DROP FOREIGN KEY `FK_resources_uploads_RelateId`;
ALTER TABLE `resources_uploads` ADD CONSTRAINT `FK_resources_uploads_RelateId` FOREIGN KEY (`RelateId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;

ALTER TABLE `resources_uploads` DROP FOREIGN KEY `FK_resources_uploads_UploadId`;
ALTER TABLE `resources_uploads` ADD CONSTRAINT `FK_resources_uploads_UploadId` FOREIGN KEY (`UploadId`) REFERENCES `resources` (`Id`) ON DELETE CASCADE;
```

Далее необходимо убрать из файла `build-com.example.storage` инструкции, отвечающие за пересоздание базы данных. И раскомментировать инструкции, коотрые позволяют сервису запускаться в фоновом процессе. Для этого отредактируйте файл `build-com.example.storage` и приведите его в следующий вид:

Разместите с домашней папке администратора системы сценарий загрузки и установки:

`~/scripts/build-com.example.storage`

```
#!/bin/bash

SERVICE='com.example.storage'
PROJECT_DIR="/var/projects/$SERVICE"
WEBAPP_DIR="/opt/$SERVICE"
GIT_PATH='https://github.com/ntr1x/ntr1x-storage-launch.git'
GIT_BRANCH='master'
MAVEN='/usr/bin/mvn'
MAVEN_LOCAL='~/.m2/repository'
CONFIG_FILE="~/scripts/config-$SERVICE/application-prod.properties"

rm -rf $PROJECT_DIR
mkdir -p $PROJECT_DIR
chmod 775 $PROJECT_DIR
git clone -b $GIT_BRANCH $GIT_PATH $PROJECT_DIR
rm -rf $PROJECT_DIR/.git
cd $PROJECT_DIR
chgrp -R developers .

screen -S $SERVICE -X quit

rm -rf $WEBAPP_DIR/*
mkdir -p $WEBAPP_DIR
chmod 775 $WEBAPP_DIR
cp -R $PROJECT_DIR/. $WEBAPP_DIR
cp $CONFIG_FILE $WEBAPP_DIR/src/main/resources/
cd $WEBAPP_DIR
chgrp -R developers .
rm -rf $MAVEN_LOCAL/com/ntr1x/storage

screen -S $SERVICE -d -m $MAVEN spring-boot:run -Drun.profiles=prod -P build
#$MAVEN spring-boot:run -Drun.profiles=prod,recreate,setup -P build
#$MAVEN spring-boot:run -Drun.profiles=prod -P build
```

Можно совсем убрать закомментированные строки, или оставить их на случай,
если позже возникнет необходимость пересоздать базу данных.
Выполните этот сценарий командой:

```
sh ~/scripts/build-com.example.storage
```

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
