# Projet Holodeck — Documentation technique

Déploiement d'une infrastructure Serveur / Client virtualisée — domaine **starfleet.lan**

Dépôt du projet : `https://github.com/Andy-pauriac/holodeck`

---

## Sommaire

1. [Contexte et objectifs](#1-contexte-et-objectifs)
2. [Architecture des machines virtuelles](#2-architecture-des-machines-virtuelles)
3. [Configuration réseau : DHCP et DNS](#3-configuration-réseau--dhcp-et-dns)
4. [Serveur Web : Nginx et cohabitation PHP 7 / PHP 8](#4-serveur-web--nginx-et-cohabitation-php-7--php-8)
5. [Base de données : MariaDB et phpMyAdmin](#5-base-de-données--mariadb-et-phpmyadmin)
6. [Certificat SSL](#6-certificat-ssl-mutualisé)
7. [Serveur FTP sécurisé](#7-serveur-ftp-sécurisé-vsftpd)
8. [Authentification centralisée : LDAP](#8-authentification-centralisée--ldap)
9. [Administration de la VM : Cockpit](#9-administration-de-la-vm--cockpit-adminstarfleetlan)
10. [Politique de sécurité : pare-feu et absence de sudo](#10-politique-de-sécurité--pare-feu-et-absence-de-sudo)
11. [Pour aller plus loin](#11-pour-aller-plus-loin)
12. [Portabilité / changement d'environnement](#12-portabilité--changement-denvironnement)
13. [Synthèse des incidents rencontrés](#13-synthèse-des-incidents-rencontrés-et-résolutions)
14. [Récapitulatif des sources](#14-récapitulatif-des-sources)

---

## 1. Contexte et objectifs

Mise en place d'une infrastructure virtualisée sous VMware composée de deux machines virtuelles Debian :
- une **VM Serveur** (rôle infrastructure : réseau, web, base de données, fichiers, annuaire, administration),
- une **VM Cliente** (poste utilisateur avec interface graphique et navigateur).

L'ensemble des services est rattaché au domaine interne **starfleet.lan**.

> **Contrainte de sécurité** : aucun compte utilisateur ne dispose de `sudo`, conformément aux directives Starfleet. Toute administration système passe par le compte `root` (connexion directe via `su -`).

---

## 2. Architecture des machines virtuelles

### 2.1 VM Serveur

| Paramètre | Valeur |
|---|---|
| Système | Debian (base, sans interface graphique) |
| RAM | 2 Go |
| vCPU | 2 |
| Disque | 32 Go |
| Réseau | 2 cartes réseau : **WAN** (`ens34`, sortie Internet, IP dynamique), **LAN** (`ens33`, IP statique `192.168.10.1/24`, réseau interne starfleet.lan) |

### 2.2 VM Cliente

| Paramètre | Valeur |
|---|---|
| Système | Debian (avec interface graphique — XFCE) |
| RAM | 2 Go |
| vCPU | 2 |
| Disque | 16 Go |
| Réseau | 1 carte réseau, connectée au même segment LAN que la VM Serveur |
| Logiciels | Navigateur web (Firefox ESR), open-vm-tools-desktop |

La séparation WAN/LAN s'identifie via `ip a` : la carte joignable depuis l'hôte physique (utilisée pour l'administration SSH) est la **WAN** ; l'autre, sans IP par défaut, est configurée statiquement en passerelle du LAN interne.

📖 Source : [Documentation VMware Workstation Pro — Network Adapter Settings](https://docs.vmware.com/en/VMware-Workstation-Pro/index.html)

---

## 3. Configuration réseau : DHCP et DNS

### 3.1 Serveur DHCP (isc-dhcp-server)

```bash
apt install -y isc-dhcp-server
```

`/etc/default/isc-dhcp-server` :
```
INTERFACESv4="ens33"
```

`/etc/dhcp/dhcpd.conf` :
```
authoritative;

subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option domain-name-servers 192.168.10.1;
    option domain-name "starfleet.lan";
}
```

 Source : [ISC — Documentation officielle DHCP](https://www.isc.org/dhcp/)

### 3.2 Serveur DNS (BIND9)

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

Enregistrements A dans `/etc/bind/db.starfleet.lan` (ns, www8, www7, php, admin, vscore) pointant vers `192.168.10.1`.

**Résolution des domaines externes (forwarders)** — nécessaire pour que la VM Cliente accède à Internet via le LAN (mises à jour, dépôts) :

`/etc/bind/named.conf.options` :
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

Sources :
- [ISC — Documentation officielle BIND9](https://bind9.readthedocs.io/en/latest/)
- [Debian Wiki — Bind9](https://wiki.debian.org/Bind9)

### 3.3 Routage LAN → WAN (NAT)

Pour que la VM Cliente (LAN uniquement) accède à Internet via la VM Serveur :

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Règle NAT (voir section pare-feu pour le script complet) :
```bash
iptables -t nat -A POSTROUTING -o ens34 -j MASQUERADE
iptables -A FORWARD -i ens33 -o ens34 -j ACCEPT
```

---

## 4. Serveur Web : Nginx et cohabitation PHP 7 / PHP 8

### 4.1 Installation de Nginx (dépôt officiel nginx.org)

```bash
wget -qO - https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian $(lsb_release -cs) nginx" > /etc/apt/sources.list.d/nginx.list
apt update && apt install -y nginx
```

> **Point d'attention** : le paquet officiel `nginx.org` fait tourner les workers sous l'utilisateur système **`nginx`**, et non `www-data` comme la convention Debian habituelle. Toute configuration de permissions (sockets PHP-FPM, fichiers servis) doit être alignée sur cet utilisateur précis.

Sources :
- [NGINX — Installing NGINX Open Source](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/)
- [nginx.org — Linux packages](https://nginx.org/en/linux_packages.html)

### 4.2 Installation de PHP 7.4 et 8.3 (dépôt Sury)

> Le dépôt **Sury** est la référence de facto pour PHP multi-version sur Debian, mais **n'est pas un dépôt officiel Debian** (Debian Wiki).

```bash
wget -qO - https://packages.sury.org/php/apt.gpg | gpg --dearmor -o /usr/share/keyrings/sury-php.gpg
echo "deb [signed-by=/usr/share/keyrings/sury-php.gpg] https://packages.sury.org/php/ $(lsb_release -cs) main" > /etc/apt/sources.list.d/php.list
apt update
apt install -y php7.4 php7.4-fpm php7.4-mysql php7.4-ldap php7.4-mbstring \
                php8.3 php8.3-fpm php8.3-mysql php8.3-ldap php8.3-mbstring
```

Chaque version tourne comme un pool **PHP-FPM** distinct (`/run/php/php7.4-fpm.sock` et `/run/php/php8.3-fpm.sock`).

**Permissions des sockets** (`/etc/php/<version>/fpm/pool.d/www.conf`) — alignées sur l'utilisateur réel de Nginx :
```
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

Sources :
- [Debian Wiki — AdditionalPHPVersions](https://wiki.debian.org/AdditionalPHPVersions)
- [packages.sury.org — Script d'installation officiel](https://packages.sury.org/php/README.txt)

### 4.3 Virtual hosts Nginx

| Sous-domaine | Service | Backend |
|---|---|---|
| `www8.starfleet.lan` | Site PHP 8 | `php8.3-fpm.sock` |
| `www7.starfleet.lan` | Site PHP 7 | `php7.4-fpm.sock` |
| `php.starfleet.lan` | phpMyAdmin | `php8.3-fpm.sock` |
| `admin.starfleet.lan` | Administration (Cockpit, reverse proxy) | — |
| `vscore.starfleet.lan` | VS Code Server (bonus) | — |

Exemple de vhost (`www8`) :
```nginx
server {
    listen 443 ssl;
    server_name www8.starfleet.lan;
    root /var/www/www8;
    index index.php index.html;
    ssl_certificate     /etc/ssl/starfleet/starfleet.crt;
    ssl_certificate_key /etc/ssl/starfleet/starfleet.key;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

Source : [NGINX — Request processing / virtual servers](https://nginx.org/en/docs/http/request_processing.html)

---

## 5. Base de données : MariaDB et phpMyAdmin

### 5.1 MariaDB (dépôt officiel, dernière version)

```bash
wget https://r.mariadb.com/downloads/mariadb_repo_setup
chmod +x mariadb_repo_setup
./mariadb_repo_setup --mariadb-server-version=latest
apt update && apt install -y mariadb-server mariadb-client
```

Source : [MariaDB — Package Repository Setup and Usage](https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/mariadb-package-repository-setup-and-usage)

### 5.2 phpMyAdmin (php.starfleet.lan)

```bash
apt install -y phpmyadmin
```

Le vhost pointe vers `/usr/share/phpmyadmin` avec le socket PHP 8.3. L'authentification se fait avec un **compte MariaDB** (ex. `root`), distinct des comptes LDAP utilisés pour le site web.

> **Points d'attention rencontrés** :
> - Le fichier `blowfish_secret.inc.php` (`/var/lib/phpmyadmin/`) et `config-db.php` (`/etc/phpmyadmin/`) doivent être lisibles par l'utilisateur `nginx` (et non `www-data`) — sinon phpMyAdmin échoue silencieusement avec un message générique `Internal error: Error`.
> - L'extension **`php-mbstring`** est indispensable à phpMyAdmin ; son absence produit la même erreur générique.

Source : [phpMyAdmin — Documentation officielle](https://docs.phpmyadmin.net/)

---

## 6. Certificat SSL (mutualisé)

Certificat auto-signé wildcard, couvrant tous les sous-domaines de `starfleet.lan` (domaine interne, non éligible à Let's Encrypt), réutilisé par Nginx et vsftpd.

```bash
mkdir -p /etc/ssl/starfleet
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/starfleet/starfleet.key \
  -out /etc/ssl/starfleet/starfleet.crt \
  -subj "/CN=*.starfleet.lan" \
  -addext "subjectAltName=DNS:*.starfleet.lan,DNS:starfleet.lan"
```

Source : [OpenSSL — openssl-req](https://docs.openssl.org/master/man1/openssl-req/)

---

## 7. Serveur FTP sécurisé (vsftpd)

```bash
apt install -y vsftpd
```

`/etc/vsftpd.conf` :
```
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
```

> **Point d'attention** : la directive `listen=YES` est indispensable pour un fonctionnement en mode autonome via systemd — son absence produit l'erreur `500 OOPS: vsftpd: not configured for standalone, must be started from inetd`.

L'authentification se fait avec les **comptes système Linux** de la VM Serveur ; chaque utilisateur est confiné (`chroot`) dans `/var/www`.

**Test réalisé** avec `lftp` en FTPES (FTP explicite sur TLS) depuis la VM Cliente, confirmant le chiffrement TLS et le chroot fonctionnels.

Source : [vsftpd — Documentation de configuration](https://security.appspot.com/vsftpd/vsftpd_conf.html)

---

## 8. Authentification centralisée : LDAP

### 8.1 Annuaire OpenLDAP

```bash
apt install -y slapd ldap-utils
dpkg-reconfigure slapd
```

Configuré avec le suffixe `dc=starfleet,dc=lan`. Une unité organisationnelle `ou=users` regroupe les comptes utilisateurs (ex. `uid=alex,ou=users,dc=starfleet,dc=lan`).

Source : [OpenLDAP — Documentation officielle](https://openldap.org/doc/)

### 8.2 Authentification applicative (formulaire PHP)

Page `login.php` (PHP 8, `www8`) réalisant un `ldap_bind()` avec les identifiants saisis, puis ouverture de session PHP (`$_SESSION`) en cas de succès.

```php
$userDn = "uid=" . ldap_escape($user, "", LDAP_ESCAPE_DN) . ",ou=users,dc=starfleet,dc=lan";
if (@ldap_bind($ldap, $userDn, $pass)) {
    $_SESSION["user"] = $user;
}
```

Source : [PHP Manual — ldap_bind()](https://www.php.net/manual/en/function.ldap-bind.php)

### 8.3 Authentification au niveau serveur (Nginx auth_request)

Dossier `/secure/` protégé par `auth_request`, délégant la vérification à un script `ldap-check.php` renvoyant `200` ou `401` selon le résultat du `ldap_bind()`.

```nginx
location = /auth/ldap-check.php {
    internal;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    fastcgi_param SCRIPT_FILENAME /var/www/auth/ldap-check.php;
    include fastcgi_params;
}

location /secure/ {
    auth_request /auth/ldap-check.php;
    try_files $uri $uri/ =404;
}
```

Source : [NGINX — auth_request module](https://nginx.org/en/docs/http/ngx_http_auth_request_module.html)

**Les deux mécanismes ont été testés avec succès** avec l'utilisateur LDAP `alex`.

---

## 9. Administration de la VM : Cockpit (admin.starfleet.lan)

```bash
apt install -y cockpit
systemctl enable --now cockpit.socket
```

Cockpit écoute nativement en HTTPS sur le port 9090 ; il est exposé via Nginx en **reverse proxy** avec support WebSocket (nécessaire à son interface) :

```nginx
location / {
    proxy_pass https://127.0.0.1:9090;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_ssl_verify off;
}
```

Source : [Cockpit Project — Documentation officielle](https://cockpit-project.org/documentation.html)

---

## 10. Politique de sécurité : pare-feu et absence de sudo

### 10.1 Absence de sudo

```bash
gpasswd -d <utilisateur> sudo
apt remove --purge sudo
```

Toute administration passe par `su -` (root).

### 10.2 Pare-feu iptables (politique DROP par défaut)

Principe du moindre privilège : tout bloquer, puis autoriser explicitement les ports requis.

```bash
IPT=iptables
WAN=ens34
LAN=ens33

$IPT -P INPUT DROP
$IPT -P FORWARD DROP
$IPT -P OUTPUT ACCEPT

$IPT -A INPUT -i lo -j ACCEPT
$IPT -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
$IPT -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Services internes (LAN uniquement)
$IPT -A INPUT -i $LAN -p tcp --dport 22 -j ACCEPT
$IPT -A INPUT -i $LAN -p udp --dport 53 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 53 -j ACCEPT
$IPT -A INPUT -i $LAN -p udp --dport 67 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 80 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 443 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 21 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 40000:40100 -j ACCEPT
$IPT -A INPUT -i $LAN -p tcp --dport 389 -j ACCEPT

# SSH également autorisé depuis le WAN (poste d'administration)
$IPT -A INPUT -i $WAN -p tcp --dport 22 -j ACCEPT

# Routage LAN -> WAN
$IPT -A FORWARD -i $LAN -o $WAN -j ACCEPT
$IPT -t nat -A POSTROUTING -o $WAN -j MASQUERADE
```

Persistance :
```bash
apt install -y iptables-persistent
netfilter-persistent save
```

> **Point d'attention** : le port SSH doit être explicitement ouvert sur l'interface WAN si l'administration se fait depuis le poste hôte via cette interface — sinon la politique DROP bloque l'administrateur lui-même.

Source : [Netfilter/iptables — Documentation officielle](https://www.netfilter.org/documentation/index.html#documentation-howto)

---

## 11. Pour aller plus loin

### 11.1 VS Code Server (vscore.starfleet.lan)

```bash
curl -fsSL https://code-server.dev/install.sh | sh
systemctl enable --now code-server@root
```

Exposé via Nginx en reverse proxy (même principe que Cockpit, avec support WebSocket), sur le port local `8080`.

Source : [code-server — Documentation officielle (Coder)](https://coder.com/docs/code-server)

### 11.2 Script de sauvegarde automatisée

Le script `backup-holodeck.sh` sauvegarde dans une archive `.tar.gz` horodatée (`/root/backups/`) :
- paramètres réseau (interfaces, routes, resolv.conf)
- configurations DHCP, DNS, Nginx, PHP (tous pools détectés dynamiquement), FTP, LDAP (export `.ldif`), SSL, pare-feu (`iptables-save`)
- dump complet des bases MariaDB
- contenu des sites web
- liste des paquets installés et services actifs (pour réinstallation à l'identique)
- **rotation automatique** des archives de plus de 14 jours

Une tâche cron (`install-cron.sh`) programme son exécution quotidienne à 3h du matin.

```bash
0 3 * * * /root/backup-holodeck.sh >> /root/backups/cron.log 2>&1
```

---

## 12. Portabilité / changement d'environnement

Le réseau LAN interne (`192.168.10.0/24`) est entièrement autonome et ne dépend d'aucune configuration réseau externe : DHCP, DNS, certificats et vhosts reposent sur des noms de domaine ou une IP fixe (`192.168.10.1`), pas sur le réseau physique de l'hôte.

En revanche, en cas de **réimportation des VM sur un autre poste physique**, les noms d'interface réseau (`ens33`/`ens34`) peuvent être renommés différemment par le nouvel hyperviseur. Il convient alors de vérifier `ip a` et d'adapter si besoin :
- `/etc/network/interfaces`
- `/etc/default/isc-dhcp-server` (`INTERFACESv4`)
- le script pare-feu (variables `WAN=`/`LAN=`)

---

## 13. Synthèse des incidents rencontrés et résolutions

| Incident | Cause / résolution |
|---|---|
| `usermod` introuvable | Absence de `/usr/sbin` dans le `PATH` d'une session `su` sans tiret (`su -` requis) |
| `isc-dhcp-server` : *Not configured to listen on any interfaces* | `INTERFACESv4` non renseigné |
| `isc-dhcp-server` : *No subnet declaration* | Bloc `subnet` absent/dupliqué dans `dhcpd.conf` |
| `nginx` : *Address already in use* (port 80) | Apache2 installé automatiquement avec `phpmyadmin`, actif en arrière-plan |
| `502 Bad Gateway` (PHP-FPM) | Socket PHP-FPM appartenant à `www-data` alors que Nginx (paquet nginx.org) tourne sous `nginx` |
| `File not found` (PHP-FPM) | Fichier PHP jamais réellement créé sur le disque malgré la commande annoncée |
| `500 Internal Server Error` (login.php) | Extension `php-ldap` non installée pour la version PHP utilisée |
| `403 Forbidden` sur `/secure/` | `index html` non déclaré dans le vhost, empêchant Nginx de servir `index.html` |
| phpMyAdmin : `Internal error: Error` | Permissions insuffisantes sur `blowfish_secret.inc.php` et `config-db.php` pour l'utilisateur `nginx` ; extension `php-mbstring` manquante |
| `vsftpd` : *not configured for standalone* | Directive `listen=YES` absente du fichier de configuration |
| SSH inaccessible après activation du pare-feu | Règle SSH limitée à l'interface LAN, alors que l'administration se faisait depuis le WAN |
| `install-cron.sh` : tâche cron non installée malgré exécution sans erreur apparente | `set -e` interrompant silencieusement le script sur l'échec de `grep` en l'absence de crontab existant |
| Installateur Debian : *miroir de l'archive corrompu* | Absence de réseau fonctionnel au moment de l'installation ; changer de miroir ou vérifier la carte réseau utilisée |

---

## 14. Récapitulatif des sources

| Sujet | Lien officiel |
|---|---|
| VMware Workstation Pro | https://docs.vmware.com/en/VMware-Workstation-Pro/index.html |
| ISC DHCP | https://www.isc.org/dhcp/ |
| ISC BIND9 | https://bind9.readthedocs.io/en/latest/ |
| Debian Wiki — Bind9 | https://wiki.debian.org/Bind9 |
| NGINX — installation officielle | https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/ |
| nginx.org — packages Linux | https://nginx.org/en/linux_packages.html |
| NGINX — server blocks | https://nginx.org/en/docs/http/request_processing.html |
| NGINX — auth_request module | https://nginx.org/en/docs/http/ngx_http_auth_request_module.html |
| Debian Wiki — AdditionalPHPVersions (Sury) | https://wiki.debian.org/AdditionalPHPVersions |
| packages.sury.org — script officiel | https://packages.sury.org/php/README.txt |
| PHP Manual — ldap_bind() | https://www.php.net/manual/en/function.ldap-bind.php |
| phpMyAdmin — documentation | https://docs.phpmyadmin.net/ |
| MariaDB — repository setup | https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/mariadb-package-repository-setup-and-usage |
| OpenSSL — openssl-req | https://docs.openssl.org/master/man1/openssl-req/ |
| vsftpd — configuration | https://security.appspot.com/vsftpd/vsftpd_conf.html |
| OpenLDAP — documentation | https://openldap.org/doc/ |
| Debian Wiki — LDAP | https://wiki.debian.org/LDAP |
| Cockpit Project — documentation | https://cockpit-project.org/documentation.html |
| code-server (Coder) — documentation | https://coder.com/docs/code-server |
| Netfilter/iptables — documentation | https://www.netfilter.org/documentation/index.html#documentation-howto |

