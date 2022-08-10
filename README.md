# Kaip įdiegti Moodle Ubuntu 22.04 server

## Reikalavimai

Reikia turėti fizinį serverį su lxc (virtualizacijos) palaikymu.

## Virtualaus serverio paruošimas

Sukuriame virtualų serverį vardu `moodle`
```
lxc launch ubuntu:22.04 moodle
```

Pažiūrim rezultatą:
```
lxc list 
```

Matome:
``` 
+---------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
|        NAME         |  STATE  |        IPV4         |                     IPV6                      |   TYPE    | SNAPSHOTS |
+---------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+

| moodle              | RUNNING | 10.83.93.120 (eth0) | fd42:95fe:619d:7c6e:216:3eff:fe6f:480e (eth0) | CONTAINER | 0         |
+---------------------+---------+---------------------+-----------------------------------------------+-----------+-----------+
```
Įsimename IPV4 adresą. Jo prireiks konfigūruojant web serverį.

Iš pagrindinio serverio prisijungiame prie naujai sukurto serverio `moodle`: 
```
lxc exec moodle bash
```

## Diegiame programas reikalingas moodle

Įdiegiame tekstinį redaktorių nano:
```
apt install nano mc 
```

Ubuntu 22.04 standartiškai yra PHP 8, o moodle veikia su PHP 7.x.
Pridedame repozitoriją, turinčia visas PHP versijas:
```
apt install software-properties-common apt-transport-https
add-apt-repository ppa:ondrej/php
apt update
```

Diegiame PHP 7.4 ir bibliotekas reikalingas moodle:
```
apt install php7.4 php7.4-common libapache2-mod-php7.4 php7.4-cli
apt install graphviz aspell ghostscript clamav php7.4-pspell php7.4-curl php7.4-gd php7.4-intl php7.4-mysql php7.4-xml php7.4-xmlrpc php7.4-ldap php7.4-zip php7.4-soap php7.4-mbstring
```

Diegiame mysql.
paprastai diegtume 
```
apt install mysql-server mysql-client
```
Kas pagal nutylėjimą sudiegtų mariaDB. Tačiau šios instrukcijos rašymo metu buvo problema su installeriu Ubuntu 22.04 versijai. Forumuose žadėjo artimiausiu metu sutvarkyti.
Sudiegiame MySQL 8.0:
``` 
apt install mysql-server-8.0 mysql-client
```
Vėliau išaiškėjo, kad tai geras sprendimas, nes jei būtų senesnė MySQL versija ar mariaDB reikėtų dar paredaguoti `/etc/mysql/mysql.conf.d/mysqld.cnf`
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
ir pridėti:
``` 
default_storage_engine = innodb
```
```
innodb_file_per_table = 1
```
innodb_file_format = Barracuda 
```
Ant MySQL 8.0 to nereikia.

Įdiegiame git, kad vėliau būtų lengviau atnaujinti moodle:
```
apt install git
```

## moodle atsisiuntimas

Pridedame userį moodle:
```
adduser moodle
```

Padarome, kad apache useris www-data galėtų pasiekti  moodle userio redaguotus failus:
```
adduser www-data moodle
```

Pereiname į userio `moodle` katalogą, jame diegsime moodle:
```
cd /home/moodle
```

Klonuojame moodle iš git:
```
git clone git://git.moodle.org/moodle.git
```

Pereiname į moodle programos katalogą
```
cd /home/moodle/moodle
```

Pasižiūrime, kokios yra versijos:
```
git branch -a
```

Pasirenkame naujausią stabilią versiją:
```
git checkout MOODLE_400_STABLE
```

Pasitikrinam ar nėra atnaujinimų
```
git pull
```

Sutvarkome leidimus failams:
```
chown -R moodle:moodle /home/moodle/moodle/
chmod -R ug+rwX  /home/moodle/moodle/
```

## konfigūruojame apache

Naudojamės Ubuntu metodika.

Atsidarome Apache konfigūracijos direktoriją
```
cd /etc/apache2/sites-available
```

Sukuriame moodle puslapio konfigūracinį failą :
```
nano MANO_KIETAS_DOMENOVARDAS.lt.conf
```
Jei bus be `.conf` neveiks!

Ten įdedame:
```
<VirtualHost *:80>
    ServerName MANO_KIETAS_DOMENOVARDAS.lt

    ServerAdmin webmaster@MANO_KIETAS_DOMENOVARDAS.lt
    DocumentRoot /home/moodle/moodle

    DirectoryIndex index.html index.htm index.php

    <Directory /home/moodle/moodle>
        Options -Indexes +IncludesNOEXEC +FollowSymLinks
        allow from all
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Užsaugome: `ctrl+x`, `y`.

Įjungiame pusalapį apache:

```
ln -s /etc/apache2/sites-available/MANO_KIETAS_DOMENOVARDAS.lt.conf  /etc/apache2/sites-enabled/MANO_KIETAS_DOMENOVARDAS.lt.conf
```
Jei bus be `.conf` neveiks!

Perkrauname apache
```
service apache restart
```

## Paruošiame MySQL

Kadangi naudojame ubuntu 22.04, root useris pasiekia mysql root be slaptažodžio.

```
mysql
```

Sukuriame duomenų bazę:
```
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Sukuriame mysql userį 
```
create user 'moodle_userio_vardas'@'localhost' IDENTIFIED BY 'super-slaptas-Slaptažodis';
```

Suteikiame teises mysql useriui:
```
GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,CREATE TEMPORARY TABLES,DROP,INDEX,ALTER ON moodle.* TO 'moodle_userio_vardas'@'localhost';
```

išeinam iš mysql su:
```
quit;
```

## Reverse proxy ir https

Grįžtame į pagrindinį serverį.
```
exit
```

Čia naudojame nginx.

### pasiruošimas gauti https sertifikatą

Atsidarome Nginx konfigūracijos direktoriją
```
cd /etc/nginx/sites-available
```

Sukuriame moodle puslapio konfigūracinį failą :
```
nano MANO_KIETAS_DOMENOVARDAS.lt
```

Sukuriame laikiną konfigą:
```
server {
    listen 80;

    server_name MANO_KIETAS_DOMENOVARDAS.lt;

    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        alias /var/www/.well-known/acme-challenge/;
    }
}
```

Įjungiame pusalapį nginx:

```
ln -s /etc/nginx/sites-available/MANO_KIETAS_DOMENOVARDAS.lt  /etc/nginx/sites-enabled/MANO_KIETAS_DOMENOVARDAS.lt
```

Perkrauname nginx
```
service nginx reload
```

Jei neturime certbot - isidiegiame:
```
apt install certbot
```

Sugeneruojame https sertifikatą iš letsencrypt: 
```
certbot certonly --webroot --webroot-path /var/www/ -d MANO_KIETAS_DOMENOVARDAS.lt
```

Vėl redaguojame moodle puslapio konfigūracinį failą :
```
nano MANO_KIETAS_DOMENOVARDAS.lt
```

Sukuriame tikrą konfigūraciją:
```
server {
    listen 80;

    server_name MANO_KIETAS_DOMENOVARDAS.lt;


    ###### letsencrypt nustatymai #####
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        alias /var/www/.well-known/acme-challenge/;
    }

    ### reikalaujame naudoti https
    return 301 https://MANO_KIETAS_DOMENOVARDAS.lt$request_uri;
}

server {

    listen 443 ssl;

    server_name MANO_KIETAS_DOMENOVARDAS.lt;

    ###### SSL nustatymai ######
        ssl_session_timeout 1h;
    ssl_session_cache shared:SSL:50m;

        keepalive_timeout    60;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    # Generate with:
    #   openssl dhparam -out /etc/nginx/dhparam.pem 2048
#    ssl_dhparam /etc/nginx/dhparam.pem;

    # What Mozilla calls "Intermediate configuration"
    # Copied from https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-D
SS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-A
ES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE
-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:D
ES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ###### letsencrypt nustatymai #####
    location ^~ /.well-known/acme-challenge/ {
        default_type "text/plain";
        alias /var/www/.well-known/acme-challenge/;
    }

    ##### PUSLAPIO NUSTATYMAI #####

    ssl_certificate /etc/letsencrypt/live/MANO_KIETAS_DOMENOVARDAS.lt/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/MANO_KIETAS_DOMENOVARDAS.lt/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/MANO_KIETAS_DOMENOVARDAS.lt/fullchain.pem;

    location /
    {
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header X-Real-IP $remote_addr;
        ## As you can see the following line should be missing and not exist in your reverse proxy nginx configuration:
        ## proxy_set_header Host $host;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host:$server_port;
        proxy_set_header X-Forwarded-Server $host;

        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        proxy_pass_request_headers      on;

        proxy_pass http://10.83.93.120:80; ## toks mano virtualaus moodle serverio visinis ip adresas

        ## "proxy_redirect default" should be placed after the "proxy_pass"
        proxy_redirect default;

#        proxy_read_timeout 900;
        proxy_connect_timeout       3000;
        proxy_send_timeout          3000;
        proxy_read_timeout          3000;
        send_timeout                3000;


        proxy_buffer_size          128k;
        proxy_buffers              4 256k;
        proxy_busy_buffers_size    256k;
    }

    client_max_body_size 100M;
}
```

Perkrauname nginx
```
service nginx reload
```

## moodle diegimas 

Toliau galime sekti [https://docs.moodle.org/39/en/Step-by-step_Installation_Guide_for_Ubuntu|Step-by-step Installation Guide for Ubuntu 20.04] nuo 7 žingsnio. Iki 7 mes darome teisingiau.



