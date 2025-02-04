FROM redhat/ubi9-init
LABEL authors = "OrangeHRM TechOps <techops@orangehrm.com>"

ARG REDHAT_USERNAME
ARG REDHAT_PASSWORD

WORKDIR /var/www/html

# Register a system with the Red Hat Customer Portal
RUN subscription-manager register --username ${REDHAT_USERNAME} --password ${REDHAT_PASSWORD} && \
subscription-manager refresh && \
subscription-manager attach --auto

RUN dnf update -y

# -----------------------------------------------------------------------------
# Install utility tools 
# -----------------------------------------------------------------------------
RUN  dnf install -y langpacks-en \
glibc-all-langpacks \
unzip \
vim \
wget \
make 

# -----------------------------------------------------------------------------
# Install Middlewares and modules
# -----------------------------------------------------------------------------

RUN dnf install -y httpd \
mod_ssl

RUN sed -i 's:^LoadModule mpm_event_module modules/mod_mpm_event.so$:#LoadModule mpm_event_module modules/mod_mpm_event.so:g' /etc/httpd/conf.modules.d/00-mpm.conf
RUN sed -i 's:^#LoadModule mpm_prefork_module modules/mod_mpm_prefork.so$:LoadModule mpm_prefork_module modules/mod_mpm_prefork.so:g' /etc/httpd/conf.modules.d/00-mpm.conf


# add epel repository
RUN dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E '%{rhel}').noarch.rpm

#Install services
RUN dnf install -y memcached \
cronie 

# Install prerequisites
RUN dnf install -y libmemcached-devel 

# Enable PHP 8.2 module
RUN dnf module reset php -y
RUN dnf module enable php:8.2 -y

# Install PHP 
RUN dnf install -y php

# Install PHP Modules
RUN dnf install -y php-ldap \
php-bcmath \
php-devel \
php-mysqlnd \
php-pdo \
php-soap \
php-gd \
php-json \
php-gmp \
php-pecl-apcu \
php-pecl-zip \
php-pear.noarch

# Install dependencies for PECL packages
RUN dnf install -y libmcrypt libmcrypt-devel libssh2-devel --disablerepo="*" --enablerepo="epel"

# Install third party PHP packages
RUN yes | pecl install -f igbinary-3.2.6 ssh2-1.3.1 mcrypt-1.0.7 && \
    printf "\n" | pecl install -f memcached-3.1.5

##Install php-tidy and php-imap modules##

# # Install prerequisites
RUN  dnf install -y diffutils \
libtidy-devel

# # Download and extract the PHP source code
RUN mkdir ~/php_source 
RUN cd ~/php_source && \
wget https://www.php.net/distributions/php-$(php -r 'echo PHP_VERSION;').tar.gz && \
gunzip php-$(php -r 'echo PHP_VERSION;').tar.gz &&\
tar xvf php-$(php -r 'echo PHP_VERSION;').tar

# # Build and enable the tidy module
RUN cd ~/php_source/php-$(php -r 'echo PHP_VERSION;')/ext/tidy && \
phpize && \
./configure && \
make
RUN cd ~/php_source/php-$(php -r 'echo PHP_VERSION;')/ext/tidy && \
mv modules/tidy.so $(php -r 'echo ini_get("extension_dir");') && \
echo "extension=tidy.so" >> /etc/php.d/40-tidy.ini

#Build and enable the imap module
# RUN cd ~/php_source/php-$(php -r 'echo PHP_VERSION;')/ext/imap && \
# phpize && \
# ./configure --with-libdir=lib64 --with-kerberos --with-imap-ssl &&\
# make
# RUN cd ~/php_source/php-$(php -r 'echo PHP_VERSION;')/ext/imap && \
# mv modules/imap.so $(php -r 'echo ini_get("extension_dir");') && \
# echo "extension=imap.so" >> /etc/php.d/40-imap.ini

# #Reload apache service and remove the downloaded source code
RUN rm -rf ~/php_source

# Update the default apache site with the config we created.
RUN mkdir -p /etc/httpd/sites-available /etc/httpd/sites-enabled && \
    echo "IncludeOptional sites-enabled/*.conf" >> /etc/httpd/conf/httpd.conf && \
    ln -s /etc/httpd/sites-available/orangehrm.conf /etc/httpd/sites-enabled/
COPY apache-config.conf /etc/httpd/sites-available/orangehrm.conf


# Copy IonCube loader and PHP configuration files
COPY ioncube/ioncube_loader_lin_8.2.so /usr/lib64/php/modules/ioncube_loader_lin_8.2.so
COPY php.ini /etc/php.ini

# Copy Cert and Private Keys
COPY httpd/cert/localhost.crt /etc/pki/tls/certs/localhost.crt
COPY httpd/cert/localhost.key /etc/pki/tls/private/localhost.key

# Install Other Server Dependencies
RUN dnf install -y ImageMagick \
openssl-devel \ 
poppler-utils \
libreoffice-draw \
libreoffice-writer \
urw-base35-fonts-20200910-6.el9.noarch

RUN dnf group install -y "Fonts"

# RUN wget https://alge.orangehrm.com/downloads/ohrm-font-installer-rhel && \
# unzip ohrm-font-installer-rhel && \
# cd font-installer && \
# bash install-fonts.sh

## Install MariaDB-client ##

# Reset the MariaDB module
RUN dnf module reset mariadb -y

# Enable the MariaDB repository
RUN cd ~ && \ 
curl -LsS -O https://downloads.mariadb.com/MariaDB/mariadb_repo_setup && \
bash mariadb_repo_setup --mariadb-server-version=10.11.7 --skip-check-installed

# Refresh the repository cache
RUN dnf makecache

# # Install MariaDB from the new repository
RUN dnf repolist && \
dnf -y install MariaDB-client

# Install QPDF Library
COPY ./scripts/qpdf-install.sh .
RUN chmod +x qpdf-install.sh
RUN ./qpdf-install.sh
RUN rm -rf qpdf-install.sh

# Unregister subscription
RUN subscription-manager unregister

#Update Oslon Timezone Database
RUN pecl install timezonedb

#Generate report on the installed packages
RUN rpm -qa --queryformat "%{NAME} %{VERSION}-%{RELEASE} %{ARCH} %{INSTALLTIME:date} \n" > /var/installed_packages_detailed_report.txt

# Enable services
RUN systemctl enable httpd.service
RUN systemctl enable memcached.service
RUN systemctl enable crond.service

