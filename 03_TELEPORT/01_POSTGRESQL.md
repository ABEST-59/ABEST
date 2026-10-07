# Teleport : Installation & Configuration Serveur PostgreSQL 18

Ce document décrit la procédure étape par étape pour installer et configurer un serveur PostgreSQL 18 intégré à un domaine Active Directory, sécurisé via TLS/SSL (Certbot OVH) et préparé pour une connexion avec **Teleport**.

---

## Table des matières
1. [Préparation du serveur](#1-préparation-du-serveur)
2. [Intégration Active Directory (realmd)](#2-intégration-active-directory-realmd)
3. [Installation PostgreSQL 18](#3-installation-postgresql-18)
4. [Configuration de la base de données Teleport](#4-configuration-de-la-base-de-données-teleport)
5. [Configuration du Firewall](#5-configuration-du-firewall)
6. [Génération du certificat SSL (Certbot + OVH)](#6-génération-du-certificat-ssl-certbot--ovh)
7. [Installation et permissions des certificats](#7-installation-et-permissions-des-certificats)
8. [Configuration de `postgresql.conf`](#8-configuration-de-postgresqlconf)
9. [Configuration de `pg_hba.conf`](#9-configuration-de-pg_hbaconf)
10. [Redémarrage des services](#10-redémarrage-des-services)
11. [Test de connexion TLS](#11-test-de-connexion-tls)

---

## 1. Préparation du serveur

Mettre à jour l'ensemble du système :

```bash
sudo dnf update -y
```

---

## 2. Intégration Active Directory (realmd)

### Installation des paquets requis
```bash
sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation authselect-compat -y
```

### Jonction au domaine
```bash
sudo realm join ldap.abest.ovh -U Administrator --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=ABEST DSI,DC=ldap,DC=abest,DC=ovh"
```

### Autorisations d'accès SSH
Restreindre les accès SSH au groupe Active Directory dédié :
```bash
sudo realm permit -g SSHSRVADM
```

### Attribution des droits sudo
Ajouter la règle suivante dans le fichier sudoers via `sudo visudo` ou dans `/etc/sudoers.d/ad_admins` :
```text
%SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL
```

---

## 3. Installation PostgreSQL 18

### Ajout du dépôt officiel et désactivation du module par défaut
```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
```

### Installation du serveur et des extensions
```bash
sudo dnf install -y postgresql18-server postgresql18-contrib
sudo dnf install -y wal2json_18
```

### Initialisation et démarrage de la base de données
```bash
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
sudo systemctl enable --now postgresql-18
```

---

## 4. Configuration de la base de données Teleport

Connectez-vous à PostgreSQL en tant qu'utilisateur `postgres` (`sudo -u postgres psql`) puis exécutez les requêtes suivantes :

```sql
CREATE USER teleport WITH PASSWORD '123456789AD';
CREATE DATABASE teleportdb OWNER teleport;
GRANT ALL PRIVILEGES ON DATABASE teleportdb TO teleport;
ALTER ROLE teleport WITH LOGIN REPLICATION;
```

> **Note de sécurité :** Pensez à remplacer le mot de passe par défaut par un mot de passe robuste avant le déploiement en production.

---

## 5. Configuration du Firewall

Autoriser le port par défaut de PostgreSQL (5432/TCP) dans `firewalld` :

```bash
sudo firewall-cmd --add-port=5432/tcp --permanent
sudo firewall-cmd --reload
```

---

## 6. Génération du certificat SSL (Certbot + OVH)

### Création du fichier de clés d'API OVH `/root/ovh.ini`
> ⚠️ **Avertissement :** Ne commitez pas vos véritables clés API sur un dépôt Git public !

```ini
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = 85f85d4445ef1a68
dns_ovh_application_secret = cec1b03a85a8bdc1a0d35f6c96d77479
dns_ovh_consumer_key = df3702b9926a4bf363f2cdf5668d3319
```

Appliquer les permissions de restriction sur le fichier d'identifiants :
```bash
sudo chmod 600 /root/ovh.ini
```

### Génération du certificat Let's Encrypt
```bash
sudo certbot certonly --dns-ovh --dns-ovh-credentials /root/ovh.ini -d postgres-bastion.abest.ovh
```

---

## 7. Installation et permissions des certificats

Copier les certificats générés vers le répertoire de données de PostgreSQL 18 :

```bash
sudo cp /etc/letsencrypt/live/postgres-bastion.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/server.crt
sudo cp /etc/letsencrypt/live/postgres-bastion.abest.ovh/privkey.pem /var/lib/pgsql/18/data/server.key
sudo cp /etc/letsencrypt/live/postgres-bastion.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/root.crt
```

Ajuster le propriétaire et les permissions d'accès :

```bash
sudo chown postgres:postgres /var/lib/pgsql/18/data/server.*
sudo chmod 600 /var/lib/pgsql/18/data/server.key
sudo chmod 644 /var/lib/pgsql/18/data/server.crt /var/lib/pgsql/18/data/root.crt
```

---

## 8. Configuration de `postgresql.conf`

Éditer le fichier `/var/lib/pgsql/18/data/postgresql.conf` et appliquer les paramètres suivants :

```ini
# --- Configuration SSL / TLS ---
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'

# --- Configuration Réseau & Réplication ---
listen_addresses = '*'
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
output_plugin_libraries = 'wal2json'
```

---

## 9. Configuration de `pg_hba.conf`

Éditer le fichier `/var/lib/pgsql/18/data/pg_hba.conf` pour restreindre les connexions sécurisées :

```text
# Configuration Teleport
hostssl    teleportdb    teleport    10.10.40.12/32    scram-sha-256
hostssl    teleportdb    teleport    10.10.40.12/32    cert

# Tests
#hostssl   teleportdb    teleport    10.10.50.12/32    cert
#hostssl   teleportdb    teleport    10.10.50.12/32    scram-sha-256
```

---

## 10. Redémarrage des services

Redémarrer le service PostgreSQL pour prendre en compte l'intégralité des configurations (TLS, réplication, accès network) :

```bash
sudo systemctl restart postgresql-18
```

---

## 11. Test de connexion TLS

Tester la connexion sécurisée TLS avec vérification stricte du certificat :

```bash
psql 'postgres://teleport:123456789AD@postgres-bastion.abest.ovh:5432/teleportdb?sslmode=verify-full'
```
