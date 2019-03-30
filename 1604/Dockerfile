FROM phusion/baseimage:0.11
MAINTAINER Matthew Rayner <hello@rayner.io>
ENV REFRESHED_AT 2019-03-12

# based on dgraziotin/lamp
# MAINTAINER Daniel Graziotin <daniel@ineed.coffee>

ENV DOCKER_USER_ID 501
ENV DOCKER_USER_GID 20

ENV BOOT2DOCKER_ID 1000
ENV BOOT2DOCKER_GID 50

ENV PHPMYADMIN_VERSION=4.8.5

# Tweaks to give Apache/PHP write permissions to the app
RUN usermod -u ${BOOT2DOCKER_ID} www-data && \
    usermod -G staff www-data && \
    useradd -r mysql && \
    usermod -G staff mysql

RUN groupmod -g $(($BOOT2DOCKER_GID + 10000)) $(getent group $BOOT2DOCKER_GID | cut -d: -f1)
RUN groupmod -g ${BOOT2DOCKER_GID} staff

# Install packages
ENV DEBIAN_FRONTEND noninteractive
RUN add-apt-repository -y ppa:ondrej/php && \
  apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 4F4EA0AAE5267A6C && \
  apt-get update && \
  apt-get -y upgrade && \
  apt-get -y install supervisor wget git apache2 php-xdebug libapache2-mod-php mysql-server php-mysql pwgen php-apcu php7.1-mcrypt php-gd php-xml php-mbstring php-gettext zip unzip php-zip curl php-curl && \
  apt-get -y autoremove && \
  echo "ServerName localhost" >> /etc/apache2/apache2.conf

# needed for phpMyAdmin
RUN ln -s /etc/php/7.1/mods-available/mcrypt.ini /etc/php/7.3/mods-available/ && \
  phpenmod mcrypt

# Add image configuration and scripts
ADD supporting_files/start-apache2.sh /start-apache2.sh
ADD supporting_files/start-mysqld.sh /start-mysqld.sh
ADD supporting_files/run.sh /run.sh
RUN chmod 755 /*.sh
ADD supporting_files/supervisord-apache2.conf /etc/supervisor/conf.d/supervisord-apache2.conf
ADD supporting_files/supervisord-mysqld.conf /etc/supervisor/conf.d/supervisord-mysqld.conf
ADD supporting_files/mysqld_innodb.cnf /etc/mysql/conf.d/mysqld_innodb.cnf

# Allow mysql to bind on 0.0.0.0
RUN sed -i "s/.*bind-address.*/bind-address = 0.0.0.0/" /etc/mysql/my.cnf

# Set PHP timezones to Europe/London
RUN sed -i "s/;date.timezone =/date.timezone = Europe\/London/g" /etc/php/7.3/apache2/php.ini
RUN sed -i "s/;date.timezone =/date.timezone = Europe\/London/g" /etc/php/7.3/cli/php.ini

# Remove pre-installed database
RUN rm -rf /var/lib/mysql

# Add MySQL utils
ADD supporting_files/create_mysql_users.sh /create_mysql_users.sh
RUN chmod 755 /*.sh

# Add phpmyadmin
RUN wget -O /tmp/phpmyadmin.tar.gz https://files.phpmyadmin.net/phpMyAdmin/${PHPMYADMIN_VERSION}/phpMyAdmin-${PHPMYADMIN_VERSION}-all-languages.tar.gz
RUN tar xfvz /tmp/phpmyadmin.tar.gz -C /var/www
RUN ln -s /var/www/phpMyAdmin-${PHPMYADMIN_VERSION}-all-languages /var/www/phpmyadmin
RUN mv /var/www/phpmyadmin/config.sample.inc.php /var/www/phpmyadmin/config.inc.php

# Add composer
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \
    php composer-setup.php && \
    php -r "unlink('composer-setup.php');" && \
    mv composer.phar /usr/local/bin/composer

ENV MYSQL_PASS:-$(pwgen -s 12 1)
# config to enable .htaccess
ADD supporting_files/apache_default /etc/apache2/sites-available/000-default.conf
RUN a2enmod rewrite

# Configure /app folder with sample app
RUN mkdir -p /app

#Environment variables to configure php
ENV PHP_UPLOAD_MAX_FILESIZE 10M
ENV PHP_POST_MAX_SIZE 10M

ENV TZ=America/New_York
ENV UID=10000
ENV GID=1002

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone \
    && apt-get update && apt-get -y --no-install-recommends install sudo curl xz-utils wget git python build-essential \
    && apt install -y --no-install-recommends ca-certificates apt-transport-https ruby-full \
    && curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash - \
    && apt-get install -y nodejs

RUN addgroup --gid $GID theia \
    && adduser --disabled-password --gecos '' --uid $UID --gid $GID theia \
    && adduser theia sudo \
    && echo '%sudo ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers \
    && chmod g+rw /home \
    && mkdir -p /home/project \
    && mkdir -p /app \
    && chown -R theia:theia /app \
    && chown -R theia:theia /home/theia \
    && chown -R theia:theia /home/project

WORKDIR /home/theia
COPY package.json ./package.json

RUN npx yarn --cache-folder ./ycache \
    && rm -rf ./ycache \
    && NODE_OPTIONS="--max_old_space_size=4096" npx yarn theia build

RUN apt-get install rsync php7.3-bz2 -y
# Disabled in 7.2+, but still present in this image
RUN rm /etc/php/7.3/cli/conf.d/20-mcrypt.ini

# Setup template project
RUN composer create-project --no-interaction acquia/lightning-project /home/project

EXPOSE 80 3306 3000
# VOLUME  ["/etc/mysql", "/var/lib/mysql"]

ENV SHELL /bin/bash
ENV APACHE_ROOT "project/docroot"

RUN ln -s /home/project /app/project
RUN rm -rf /var/www/html && ln -s "/app/${APACHE_ROOT}" /var/www/html

RUN echo "/run.sh & npx yarn theia start /home/project --hostname=0.0.0.0" > /newrun.sh

# Make drupal work now HACKY
RUN cp /home/project/docroot/sites/default/default.settings.php /home/project/docroot/sites/default/settings.php
RUN find /home/project/ -type d -exec chmod 777 {} \;
RUN chmod 777 /home/project/docroot/sites/default/settings.php
RUN sed -i -e 's/max_execution_time = 30/max_execution_time = 120/g' /etc/php/7.3/apache2/php.ini

CMD ["sh", "/newrun.sh"]
