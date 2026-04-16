
Перед обновлением необходимо убедиться в соответствии требований, их можно взять на официальном сайте `https://phpipam.net/documents/installation/` или на официальном репозитории `https://phpipam.net/documents/installation/`

Для текущей версии необходимы PHP от 7.2 до 8.3 и БД MySQL 8.0+ и MariaDB 10.2.1+. Текущие и доступные для обновления версии можно посмотреть с помощью команды `apt-cache policy php mysql-common mariadb-common` или целевыми командами например `php -v`, также может быть полезно проверить установлены ли необходимые компоненты php, например `gd, mysql, pdo, gmp или mbstring` командой `php -m`. 

После обновления необходимых пакетов проверяем способ установки текущей версии phpipam, для этого ищем каталог `.git`:

`ls -la /var/www/phpipam | grep .git`

Если каталога `.git` нет, значит установка проводилась вручную. Если папка есть будем обновляться с помощью `git`, как сказано в документации или п.3.

# 1. Ручной метод обновления

Для начала скачаем архив новой версии с github вручную или с помощью утилиты `wget` и распакуем содержимое в соседний каталог:

```
$ cd /tmp
# Скачиваем архив
$ wget https://github.com/phpipam/phpipam/releases/download/v1.7.4/phpipam-v1.7.4.tgz

# Распаковываем содержимое во временный каталог
$ tar -xvf phpipam-v1.7.4.tgz

# Меняем местами текущую версию и новую
$ mv /var/www/phpipam /var/www/phpipam_old
$ mv /tmp/phpipam /var/www/phpipam

# Возвращаем прежнюю конфигурацию и выдаем права
$ cp /var/www/phpipam_old/config.php /var/www/phpipam/config.php
$ chown -R www-data:www-data /var/www/phpipam
$ chmod -R 755 /var/www/phpipam

# Перезапускаем службы
$ systemctl restart nginx.service php8.x-fpm
```

# 2. Переход с ручной установки на git

Если текущая версия phpipam была установлена вручную из архива и мы для дальнейших обновлений хотим перейти на git версию, то подготовим чистый git-каталог

Чтобы не смешивать старые архивные файлы с новыми, мы скачаем проект в соседнюю папку.

```
$ apt install git -y

$ cd /var/www 
$ git clone -b master https://github.com/phpipam/phpipam.git phpipam-git

$ cd /var/www/phpipam-git
$ git submodule update --init --recursive

# Возвращаем прежнюю конфигурацию и выдаем права
$ cp /var/www/phpipam/config.php /var/www/phpipam-git/config.php

# Меняем местами текущую версию и новую и выдаем права
$ mv phpipam phpipam_old_manual
$ mv phpipam-git phpipam
$ chown -R www-data:www-data /var/www/phpipam
$ chmod -R 755 /var/www/phpipam

# Перезапускаем службы
$ systemctl restart nginx.service php8.x-fpm
```

# 3. Обновление для git версии

Если мы используем данную версию, то обновление пройдет очень просто, для этого необходимо лишь:

```
$ cd /var/www/phpipam

# Смотрим какую ветку используем, должна быть master
$ git branch -r
$ git branch

# Если не master то выбираем ее
$ git checkout master

# Получаем новый код
$ git pull

# Обновляем библиотеки 
$ git submodule update --init --recursive

# Перезапускаем службы
$ systemctl restart nginx.service php8.x-fpm
```

# Обновление базы данных

 phpIPAM умеет обновлять структуру БД сам для этого нужно:

1. Зайти в браузере по адресу твоего IPAM.
2. Система увидит, что файлы новее, чем база, и предложит **"Upgrade phpipam database"**.
3. Выбрать **Automatic database upgrade**.
4. Нажать кнопку подтверждения.

Или как альтернатива (запасной вариант) использовать скрипт вручную для генерации SQL запросов и затем запустить их вручную:

```
$ cd /var/www/phpipam
$ php functions/upgrade_queries.php 1.7 > /tmp/upgrade.sql 
$ mysql -u [user] -p phpipam < /tmp/upgrade.sql
```
