## magento2-install

##### install magento2 on ubuntu server 20.04 EC2
---------------------------------------------------


#### [STG.01] apt repos update 및 apache2 설치
```
sudo apt update
sudo apt install apache2

sudo systemctl stop apache2.service
sudo systemctl start apache2.service
sudo systemctl enable apache2.service
```


#### [STG.02] mariadb server/client 설치
```
sudo apt-get install mariadb-server mariadb-client

sudo systemctl stop mariadb.service
sudo systemctl start mariadb.service
sudo systemctl enable mariadb.service

# 초기 root password 설정
sudo mysql_secure_installation
```


#### [STG.03] apt repos 추가
```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```


#### [STG.04] phpy8.1 + unzip 설치
```
sudo apt install php8.1 libapache2-mod-php8.1 php8.1-common php8.1-gmp php8.1-curl php8.1-soap php8.1-bcmath php8.1-intl php8.1-mbstring php8.1-xmlrpc php8.1-mcrypt php8.1-mysql php8.1-gd php8.1-xml php8.1-cli php8.1-zip php8.1-curl unzip
```

#### [STG.05] php.ini 설정

> ##### sudo vi /etc/php/8.1/apache2/php.ini

```
file_uploads = On
allow_url_fopen = On
short_open_tag = On
memory_limit = 256M
upload_max_filesize = 100M
max_execution_time = 360
```

#### [STG.06] apache2 재기동 명령어 alias 등록
```
alias res='sudo systemctl restart apache2.service' >> .bashrc
source .bashrc
```

#### [STG.07] DB 접속 및 서비스 계정 생성 (사용할 계정 정보로 설정)

> ##### sudo mysql -u root -p

```
CREATE DATABASE magento2;
CREATE USER ‘mg2user‘@’localhost’ IDENTIFIED BY 'redpolex';
GRANT ALL ON magento2.* TO 'mg2user'@`localhost` IDENTIFIED BY 'redpolex' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
```


#### [STG.08] Composer 설치
```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '55ce33d7678c5a611085589f1f3ddf8b3c52d662cd01d4ba75c0ee0459970c2200a51f492d557530c71c15d8dba01eae') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/bin/composer
```

#### [STG.09] Magento2 Download

https://magento.com/tech-resources/download 접속 후 다운로드 후 /var/www/html/magento2 폴더로 이동
![image](https://user-images.githubusercontent.com/30817824/175862000-64f4abbf-3232-49ae-b230-350347efdb34.png)

아래 명령어로 다운로드 받아도 됨
```
cd /var/www
sudo chown -R ubuntu:www-data html
cd html
mkdir magento2
cd magento2
wget https://shared-redpolex.s3.ap-northeast-2.amazonaws.com/magento-ce-2.4.4_sample_data-2022-03-16-05-13-02.zip
unzip magento-ce-2.4.4_sample_data-2022-03-16-05-13-02.zip -d .
```


#### [STG.10] 폴더 권한 설정 (cache로 인한 static page error 발생 시 아래 명령어 재실행)

```
sudo chown -R ubuntu:www-data /var/www/html/magento2/
sudo chmod -R 755 /var/www/html/magento2/
sudo chmod -R 777 /var/www/html/magento2/var/
sudo chmod -R 777 /var/www/html/magento2/pub/
sudo chmod -R 777 /var/www/html/magento2/app/etc/
sudo chmod -R 777 /var/www/html/magento2/generated/
```

##### [Private commadns...Temp]
```
cd /var/www/html/magento2

sudo find var generated vendor pub/static pub/media app/etc -type f -exec chmod g+w {} +
sudo find var generated vendor pub/static pub/media app/etc -type d -exec chmod g+ws {} +
chmod u+x bin/magento

cd /var/www/html/magento2
rm -rf composer.lock

cd /etc/apache2
sudo chown -R ubuntu:root ./*
```


#### [STG.11] Apache2 설정 (Apache2 base dir은 /etc/apache2 이다)

> ##### apache2의 enabled conf 파일들 중 불필요한 심볼릭 링크 제거
```
cd /etc/apache2/sites-enabled
rm 000-default.conf
```

> ##### vi /etc/apache2/sites-available/magento2.conf
```
<VirtualHost *:80>
   ServerAdmin admin@example.com
   DocumentRoot /var/www/html/magento2
   ServerName example.com
   ServerAlias www.example.com

   <Directory /var/www/html/magento2>
      Options Indexes FollowSymLinks MultiViews
      AllowOverride All
      Order allow,deny
      allow from all
   </Directory>

   ErrorLog ${APACHE_LOG_DIR}/error.log
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

#### [STG.12] Apache Conf enable
```
cd sites-enabled
ln -s ../sites-available/magento2.conf magento2.conf
```

#### [STG.13] mod_rewrite enable
```
cd mods-enabled/
ln -s ../mods-available/rewrite.load rewrite.load
```

#### [STG.14] php 파일 인식을 위한 설정 (위 설정으로도 안되면 파일 맨 밑에 아래 내용 추가)

> ##### vi /etc/apache2/apache2.conf
```
SetHandler application/x-httpd-php
AddType application/x-httpd-php .php .htm .html .inc
AddType application/x-httpd-php-source .phps
```


#### [STG.15] elasticsearch install (https://github.com/maisonde3cochons/kafka-monitoring-elasticstack 참고)

```
curl -LO https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-oss-7.10.2-linux-x86_64.tar.gz
tar -xzf elasticsearch-oss-7.10.2-linux-x86_64.tar.gz
rm -rf elasticsearch-oss-7.10.2-linux-x86_64.tar.gz

# Background 실행
./bin/elasticsearch -d -p elastic_pid
```

---------------------------------------

#### [STG.16] Magento 설치(admin login 시 ID/PWD는 hong/redpolAdmin)
```
cd /var/www/html/magento2/bin
./magento setup:install --base-url=http://13.124.72.233 \
--db-host=localhost --db-name=magento2 --db-user=mg2user --db-password=redpolex \
--admin-firstname=Magento --admin-lastname=User --admin-email=redpolex@yopmail.com \
--admin-user=hong --admin-password=redpolAdmin --language=en_US \
--currency=USD --use-rewrites=1 \
--search-engine=elasticsearch7 --elasticsearch-host=localhost \
--elasticsearch-port=9200
```

#### [STG.17] Multi Factor 인증 제거(enable 시 admin 로그인 시 2factor 인증이 필요하지만 mail이 발송되려면 설정이 추가 필요함)
```
cd /var/www/html/magento2/bin
./magento module:disable Magento_TwoFactorAuth
```

#### [STG.18] cache 제거/production mode로 deploy/static content deploy ...etc.

```
magento sampledata:deploy
magento setup:upgrade
magento cache:flush
magento deploy:mode:show
magento deploy:mode:set production
magento setup:static-content:deploy
```

