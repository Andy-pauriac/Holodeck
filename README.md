# Projet Holodeck — Procédure de reconstruction du serveur

Reconstruction complète de la VM Serveur, avec une contrainte particulière : **PHP 7.4 et PHP 8.3 sont installés par compilation depuis les sources (wget + tar.gz), sans passer par `apt install` pour PHP lui-même.**

> Convention réseau retenue pour cette reconstruction : **WAN = `ens33`**, **LAN = `ens34`** (à vérifier avec `ip a` avant de commencer — cette correspondance peut changer d'une VM à l'autre selon l'ordre de détection des cartes par l'hyperviseur).

---

## Sommaire

1. [Système de base](#1-système-de-base)
2. [Réseau](#2-réseau--configurer-le-lan-en-ip-statique)
3. [DHCP](#3-dhcp)
4. [DNS](#4-dns)
5. [Routage NAT](#5-routage-nat-lan--wan)
6. [Nginx](#6-nginx-dépôt-officiel)
7. [PHP 7.4 et 8.3 — compilation depuis les sources](#7-php-74-et-83--compilation-depuis-wgettargz)
8. [Certificat SSL](#8-certificat-ssl)
9. [Sites web et vhosts Nginx](#9-sites-web-et-vhosts-nginx)
10. [MariaDB](#10-mariadb-dépôt-officiel)
11. [phpMyAdmin](#11-phpmyadmin--via-wget)
12. [FTP](#12-ftp-vsftpd)
13. [LDAP](#13-ldap)
14. [Cockpit](#14-cockpit--reverse-proxy)
15. [Pare-feu](#15-pare-feu)
16. [Suppression de sudo](#16-suppression-de-sudo)
17. [Sources](#17-sources)

---

## 1. Système de base

```bash
su -
apt update && apt upgrade -y
```

## 2. Réseau — configurer le LAN en IP statique

```bash
ip a
```
Identifier la carte WAN (IP dynamique, joignable depuis l'hôte physique en SSH) et la carte LAN (généralement `DOWN`, sans IP). Pour cette reconstruction : **WAN = `ens33`**, **LAN = `ens34`**.

```bash
nano /etc/network/interfaces
```
```
auto ens34
iface ens34 inet static
    address 192.168.10.1
    netmask 255.255.255.0
```
```bash
systemctl restart networking
ip a show ens34
```

📖 Source : [Debian Wiki — NetworkConfiguration](https://wiki.debian.org/NetworkConfiguration)

## 3. DHCP

```bash
apt install -y isc-dhcp-server
```

`/etc/default/isc-dhcp-server` :
```
INTERFACESv4="ens34"
```

`/etc/dhcp/dhcpd.conf` :
```bash
tee /etc/dhcp/dhcpd.conf > /dev/null << 'EOF'
default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;
authoritative;

subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option domain-name-servers 192.168.10.1;
    option domain-name "starfleet.lan";
}
EOF
```

```bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl enable --now isc-dhcp-server
systemctl status isc-dhcp-server
```

📖 Source : [ISC — Documentation officielle DHCP](https://www.isc.org/dhcp/)

## 4. DNS

```bash
apt install -y bind9 bind9utils
```

`/etc/bind/named.conf.local` :
```
zone "starfleet.lan" {
    type master;
    file "/etc/bind/db.starfleet.lan";
};
```

`/etc/bind/db.starfleet.lan` :
```
$TTL    604800
@   IN  SOA   ns.starfleet.lan. admin.starfleet.lan. (
              1         ; Serial
         604800         ; Refresh
          86400         ; Retry
        2419200         ; Expire
         604800 )       ; Negative Cache TTL
;
@           IN  NS      ns.starfleet.lan.
ns          IN  A       192.168.10.1
www8        IN  A       192.168.10.1
www7        IN  A       192.168.10.1
php         IN  A       192.168.10.1
admin       IN  A       192.168.10.1
vscore      IN  A       192.168.10.1
```

`/etc/bind/named.conf.options` (résolution des domaines externes) :
```
options {
    directory "/var/cache/bind";
    forwarders {
        8.8.8.8;
        1.1.1.1;
    };
    dnssec-validation auto;
    listen-on { any; };
};
```

```bash
systemctl enable --now bind9
systemctl status bind9
```

📖 Source : [ISC — Documentation officielle BIND9](https://bind9.readthedocs.io/en/latest/)

## 5. Routage NAT (LAN → WAN)

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

*(les règles NAT proprement dites sont posées à l'étape 15, avec le reste du pare-feu)*

## 6. Nginx (dépôt officiel)

```bash
apt install -y wget gnupg2 ca-certificates lsb-release
wget -qO - https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian $(lsb_release -cs) nginx" > /etc/apt/sources.list.d/nginx.list
apt update && apt install -y nginx
```

> Les workers Nginx (paquet nginx.org) tournent sous l'utilisateur système **`nginx`**, pas `www-data`. Toutes les permissions de fichiers/sockets PHP-FPM ci-dessous en tiennent compte.

📖 Source : [NGINX — Installing NGINX Open Source](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)

## 7. PHP 7.4 et 8.3 — compilation depuis wget/tar.gz

**Contrainte du sujet : pas de `apt install php*`.** Les outils de compilation (compilateur, bibliothèques de développement) restent installés via `apt`, seul PHP lui-même est construit depuis ses sources officielles.

### 7.1 Outils de compilation

```bash
apt install -y build-essential pkg-config autoconf bison re2c \
  libxml2-dev libsqlite3-dev libssl-dev libcurl4-openssl-dev \
  libonig-dev libzip-dev zlib1g-dev libldap2-dev libargon2-dev \
  libreadline-dev
```

### 7.2 Téléchargement des sources

```bash
mkdir -p /usr/src/php-build && cd /usr/src/php-build
wget https://www.php.net/distributions/php-8.3.14.tar.gz
wget https://museum.php.net/php7/php-7.4.33.tar.gz
```

> PHP 7.4 étant en fin de vie (EOL), ses archives sont conservées sur `museum.php.net` plutôt que sur le site principal.

📖 Sources :
- [PHP.net — Downloads](https://www.php.net/downloads)
- [PHP Museum — Archives des anciennes versions](https://museum.php.net/)

### 7.3 Compilation de PHP 8.3

```bash
cd /usr/src/php-build
tar xzf php-8.3.14.tar.gz
cd php-8.3.14

./configure \
  --prefix=/usr/local/php8.3 \
  --with-config-file-path=/usr/local/php8.3/etc \
  --enable-fpm \
  --with-fpm-user=nginx \
  --with-fpm-group=nginx \
  --enable-mbstring \
  --with-mysqli \
  --with-pdo-mysql \
  --with-ldap \
  --with-openssl \
  --with-zlib \
  --with-curl \
  --enable-opcache

make -j$(nproc)
make install
cd ..
```

### 7.4 Compilation de PHP 7.4

```bash
tar xzf php-7.4.33.tar.gz
cd php-7.4.33

./configure \
  --prefix=/usr/local/php7.4 \
  --with-config-file-path=/usr/local/php7.4/etc \
  --enable-fpm \
  --with-fpm-user=nginx \
  --with-fpm-group=nginx \
  --enable-mbstring \
  --with-mysqli \
  --with-pdo-mysql \
  --with-ldap \
  --with-openssl \
  --with-zlib \
  --with-curl \
  --enable-opcache

make -j$(nproc)
make install
cd ..
```

📖 Source : [PHP Manual — Installation Unix/Linux](https://www.php.net/manual/en/install.unix.php)

### 7.5 Configuration FPM (sockets et permissions)

```bash
mkdir -p /run/php

cp /usr/local/php8.3/etc/php-fpm.conf.default /usr/local/php8.3/etc/php-fpm.conf
cp /usr/local/php8.3/etc/php-fpm.d/www.conf.default /usr/local/php8.3/etc/php-fpm.d/www.conf
sed -i 's|^listen = .*|listen = /run/php/php8.3-fpm.sock|' /usr/local/php8.3/etc/php-fpm.d/www.conf
cat >> /usr/local/php8.3/etc/php-fpm.d/www.conf << 'EOF'
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
EOF

cp /usr/local/php7.4/etc/php-fpm.conf.default /usr/local/php7.4/etc/php-fpm.conf
cp /usr/local/php7.4/etc/php-fpm.d/www.conf.default /usr/local/php7.4/etc/php-fpm.d/www.conf
sed -i 's|^listen = .*|listen = /run/php/php7.4-fpm.sock|' /usr/local/php7.4/etc/php-fpm.d/www.conf
cat >> /usr/local/php7.4/etc/php-fpm.d/www.conf << 'EOF'
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
EOF
```

### 7.6 Services systemd dédiés

Aucun service systemd n'est généré automatiquement par `make install` — il faut le créer manuellement pour chaque version.

```bash
tee /etc/systemd/system/php8.3-fpm.service > /dev/null << 'EOF'
[Unit]
Description=PHP 8.3 FastCGI Process Manager
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/php8.3/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php8.3/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

tee /etc/systemd/system/php7.4-fpm.service > /dev/null << 'EOF'
[Unit]
Description=PHP 7.4 FastCGI Process Manager
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/php7.4/sbin/php-fpm --nodaemonize --fpm-config /usr/local/php7.4/etc/php-fpm.conf
ExecReload=/bin/kill -USR2 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now php8.3-fpm php7.4-fpm
systemctl status php8.3-fpm php7.4-fpm
ls -l /run/php/
```

📖 Source : [freedesktop.org — systemd.service documentation](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)

### 7.7 Vérification des extensions compilées

```bash
/usr/local/php8.3/bin/php -m | grep -iE "mbstring|ldap|mysqli"
/usr/local/php7.4/bin/php -m | grep -iE "mbstring|ldap|mysqli"
```

## 8. Certificat SSL

```bash
mkdir -p /etc/ssl/starfleet
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/starfleet/starfleet.key \
  -out /etc/ssl/starfleet/starfleet.crt \
  -subj "/CN=*.starfleet.lan" \
  -addext "subjectAltName=DNS:*.starfleet.lan,DNS:starfleet.lan"
```

📖 Source : [OpenSSL — openssl-req](https://docs.openssl.org/master/man1/openssl-req/)

## 9. Sites web et vhosts Nginx

```bash
mkdir -p /var/www/www8 /var/www/www7
echo "<?php phpinfo();" > /var/www/www8/index.php
echo "<?php phpinfo();" > /var/www/www7/index.php
chown -R nginx:nginx /var/www/www8 /var/www/www7
```

```bash
tee /etc/nginx/conf.d/www8.conf > /dev/null << 'EOF'
server {
    listen 443 ssl;
    server_name www8.starfleet.lan;
    root /var/www/www8;
    index index.php index.html;
    ssl_certificate     /etc/ssl/starfleet/starfleet.crt;
    ssl_certificate_key /etc/ssl/starfleet/starfleet.key;

    location / { try_files $uri $uri/ =404; }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF

tee /etc/nginx/conf.d/www7.conf > /dev/null << 'EOF'
server {
    listen 443 ssl;
    server_name www7.starfleet.lan;
    root /var/www/www7;
    index index.php index.html;
    ssl_certificate     /etc/ssl/starfleet/starfleet.crt;
    ssl_certificate_key /etc/ssl/starfleet/starfleet.key;

    location / { try_files $uri $uri/ =404; }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF

nginx -t
systemctl enable --now nginx
```

## 10. MariaDB (dépôt officiel)

```bash
wget https://r.mariadb.com/downloads/mariadb_repo_setup
chmod +x mariadb_repo_setup
./mariadb_repo_setup --mariadb-server-version=latest
apt update && apt install -y mariadb-server mariadb-client
systemctl enable --now mariadb
```

📖 Source : [MariaDB — Package Repository Setup and Usage](https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/mariadb-package-repository-setup-and-usage)

## 11. phpMyAdmin — via wget

Installé également par archive (et non `apt install phpmyadmin`, qui tire des dépendances PHP côté dépôt Debian).

```bash
cd /usr/src
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.2/phpMyAdmin-5.2.2-all-languages.tar.gz
tar xzf phpMyAdmin-5.2.2-all-languages.tar.gz
mv phpMyAdmin-5.2.2-all-languages /usr/share/phpmyadmin
cp /usr/share/phpmyadmin/config.sample.inc.php /usr/share/phpmyadmin/config.inc.php
```

```bash
openssl rand -base64 32
```
Copier la valeur générée dans `/usr/share/phpmyadmin/config.inc.php` :
```php
$cfg['blowfish_secret'] = 'VALEUR_GENEREE_ICI';
```

```bash
chown -R nginx:nginx /usr/share/phpmyadmin
```

```bash
tee /etc/nginx/conf.d/php.conf > /dev/null << 'EOF'
server {
    listen 443 ssl;
    server_name php.starfleet.lan;
    root /usr/share/phpmyadmin;
    index index.php;
    ssl_certificate     /etc/ssl/starfleet/starfleet.crt;
    ssl_certificate_key /etc/ssl/starfleet/starfleet.key;

    location / { try_files $uri $uri/ =404; }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF
nginx -t && systemctl restart nginx
```

> Authentification phpMyAdmin par **compte MariaDB** (ex. `root`), distinct des comptes LDAP utilisés côté sites web.

📖 Source : [phpMyAdmin — Documentation officielle](https://docs.phpmyadmin.net/)

## 12. FTP (vsftpd)

```bash
apt install -y vsftpd
tee /etc/vsftpd.conf > /dev/null << 'EOF'
listen=YES

anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
local_root=/var/www

ssl_enable=YES
rsa_cert_file=/etc/ssl/starfleet/starfleet.crt
rsa_private_key_file=/etc/ssl/starfleet/starfleet.key
force_local_data_ssl=YES
force_local_logins_ssl=YES

pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
EOF
systemctl enable --now vsftpd
```

> La directive `listen=YES` est indispensable pour un fonctionnement autonome via systemd (sinon `500 OOPS: vsftpd: not configured for standalone`).

📖 Source : [vsftpd — Documentation de configuration](https://security.appspot.com/vsftpd/vsftpd_conf.html)

## 13. LDAP

```bash
apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```
Domaine : `starfleet.lan` — organisation : `Starfleet` — noter le mot de passe admin défini.

```bash
slapcat -n 0 | grep olcSuffix
```
Doit afficher `dc=starfleet,dc=lan`.

📖 Source : [OpenLDAP — Documentation officielle](https://openldap.org/doc/)

## 14. Cockpit + reverse proxy

```bash
apt install -y cockpit
systemctl enable --now cockpit.socket

tee /etc/nginx/conf.d/admin.conf > /dev/null << 'EOF'
server {
    listen 443 ssl;
    server_name admin.starfleet.lan;
    ssl_certificate     /etc/ssl/starfleet/starfleet.crt;
    ssl_certificate_key /etc/ssl/starfleet/starfleet.key;

    location / {
        proxy_pass https://127.0.0.1:9090;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_ssl_verify off;
    }
}
EOF
nginx -t && systemctl restart nginx
```

📖 Source : [Cockpit Project — Documentation officielle](https://cockpit-project.org/documentation.html)

## 15. Pare-feu

Politique par défaut **DROP**, autorisation explicite des seuls ports nécessaires (principe du moindre privilège).

```bash
apt install -y iptables iptables-persistent

IPT=iptables
WAN=ens33
LAN=ens34

$IPT -P INPUT DROP
$IPT -P FORWARD DROP
$IPT -P OUTPUT ACCEPT
$IPT -A INPUT -i lo -j ACCEPT
$IPT -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
$IPT -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

$IPT -A INPUT -i $LAN -p tcp --dport 22 -j ACCEPT
$IPT -A INPUT -i $WAN -p tcp --dport 22 -j ACCEPT
$IPT -A INPUT -i $LAN -p udp --dport 53 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 53 -j ACCEPT
$IPT -A INPUT -i $LAN -p udp --dport 67 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 80 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 443 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 21 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 40000:40100 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 389 -j ACCEPT

$IPT -A FORWARD -i $LAN -o $WAN -j ACCEPT
$IPT -t nat -A POSTROUTING -o $WAN -j MASQUERADE

netfilter-persistent save
```

> Le SSH est ouvert **sur les deux interfaces** : administration possible depuis le LAN et depuis le WAN (poste hôte). Une politique DROP appliquée sans cette double règle bloquerait l'administrateur lui-même.

📖 Source : [Netfilter/iptables — Documentation officielle](https://www.netfilter.org/documentation/index.html#documentation-howto)

## 16. Suppression de sudo

Effectuée en tout dernier, une fois l'ensemble des services validés (toute correction ultérieure nécessite `su -`).

```bash
gpasswd -d <utilisateur> sudo
```

## 17. Sources

| Sujet | Lien officiel |
|---|---|
| Debian Wiki — configuration réseau | https://wiki.debian.org/NetworkConfiguration |
| ISC DHCP | https://www.isc.org/dhcp/ |
| ISC BIND9 | https://bind9.readthedocs.io/en/latest/ |
| NGINX — installation officielle | https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/ |
| PHP.net — Downloads | https://www.php.net/downloads |
| PHP Museum — archives anciennes versions | https://museum.php.net/ |
| PHP Manual — Installation Unix/Linux | https://www.php.net/manual/en/install.unix.php |
| systemd — service unit documentation | https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html |
| OpenSSL — openssl-req | https://docs.openssl.org/master/man1/openssl-req/ |
| MariaDB — repository setup | https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/mariadb-package-repository-setup-and-usage |
| phpMyAdmin — documentation | https://docs.phpmyadmin.net/ |
| vsftpd — configuration | https://security.appspot.com/vsftpd/vsftpd_conf.html |
| OpenLDAP — documentation | https://openldap.org/doc/ |
| Cockpit Project — documentation | https://cockpit-project.org/documentation.html |
| Netfilter/iptables — documentation | https://www.netfilter.org/documentation/index.html#documentation-howto |

---

## Auteur

- Prénom Nom
