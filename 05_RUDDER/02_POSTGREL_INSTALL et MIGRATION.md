# Guide de Déploiement, Migration et Sécurisation SSL de PostgreSQL 18 pour Rudder

Ce document détaille l'ensemble des procédures étape par étape pour déployer un serveur de base de données dédié sous PostgreSQL 18 (`plbdd0`), l'intégrer au domaine Active Directory, sécuriser ses accès distants avec Teleport et des certificats SSL Let's Encrypt, migrer la base de données Rudder et imposer des connexions chiffrées (`hostssl`).

## Vue d'ensemble technique

* **Serveur de base de données :** `plbdd0` / `postgres-rudder.abest.ovh` (IP : `10.10.40.10` / Serveur cible)
* **Serveur d'application :** `plrudder`
* **Domaine Active Directory :** `ldap.abest.ovh`
* **Version PostgreSQL :** 18
* **Nom de la base de données :** `rudder`
* **Utilisateur de la base de données :** `rudder`

---

## Étape 1 : Configuration initiale du système et du réseau

1. **Configuration du nom d'hôte et des interfaces réseau**

   ```bash
   vi /etc/hosts
   vi /etc/hostname
   vi /etc/NetworkManager/system-connections/ens18.nmconnection
   reboot
   ```

---

## Étape 2 : Intégration au domaine Active Directory

1. **Installation des paquets d'intégration Active Directory / LDAP**

   ```bash
   sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation authselect-compat -y
   ```

2. **Jonction au domaine et restriction des accès SSH**

   ```bash
   sudo realm join ldap.abest.ovh -U Administrator --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=DSI,OU=ABEST,DC=ldap,DC=abest,DC=ovh"
   sudo realm permit -g SSHSRVADM
   vi -f /etc/sudoers.d/domain_admins
   # AD group (prefix with %)
   %SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL
   ```

---

## Étape 3 : Installation et configuration de l'agent Teleport

1. **Installation du nœud Teleport**

   ```bash
   sudo dnf install https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm -y
   ```

2. **Configuration du nœud Teleport**

   ```bash
   teleport node configure \
     --token fd9f2aff79342d26b5de26da0f57873a \
     --ca-pin sha256:bf8a6db4179690ed87ba788089818ab4c5ac12c2d3e352a93554b256aed4668d \
     --proxy bastion.abest.ovh:443 \
     --labels env=PRD \
     --output /etc/teleport.yaml
   ```

3. **Activation et démarrage du service Teleport**

   ```bash
   sudo systemctl enable --now teleport
   ```

---

## Étape 4 : Génération du certificat SSL (Certbot + OVH DNS)

1. **Installation de Certbot et du plugin DNS OVH**

   ```bash
   sudo dnf install -y certbot python3-certbot-dns-ovh
   ```

2. **Configuration des identifiants API OVH**

   ```bash
   vi /root/ovh.ini
   chmod 600 /root/ovh.ini
   ```

3. **Génération du certificat SSL pour `postgres-rudder.abest.ovh`**

   ```bash
   certbot certonly --dns-ovh --dns-ovh-credentials /root/ovh.ini -d postgres-rudder.abest.ovh
   ```

---

## Étape 5 : Installation et configuration initiale de PostgreSQL 18

1. **Ajout du dépôt PGDG et désactivation du module PostgreSQL par défaut**

   ```bash
   sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   sudo dnf -qy module disable postgresql
   ```

2. **Installation des paquets PostgreSQL 18**

   ```bash
   sudo dnf install -y postgresql18-server postgresql18-contrib
   ```

3. **Initialisation de la base de données et activation du service**

   ```bash
   sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
   sudo systemctl enable --now postgresql-18
   ```

4. **Vérification de l'installation et des chemins de configuration**

   ```bash
   sudo -u postgres psql -c "select version();"
   sudo -u postgres psql -c "SHOW config_file;"
   sudo -u postgres psql -c "SHOW hba_file;"
   ```

---

## Étape 6 : Configuration du pare-feu (Firewalld)

1. **Affectation de l'interface à la zone interne et ouverture du port 5432**

   ```bash
   firewall-cmd --permanent --zone=internal --change-interface=ens18
   firewall-cmd --permanent --zone=internal --add-port=5432/tcp
   firewall-cmd --reload
   ```

---

## Étape 7 : Mise en place des certificats SSL dans PostgreSQL

1. **Copie des certificats Let's Encrypt dans le répertoire de données PostgreSQL**

   ```bash
   sudo cp /etc/letsencrypt/live/postgres-rudder.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/server.crt
   sudo cp /etc/letsencrypt/live/postgres-rudder.abest.ovh/privkey.pem /var/lib/pgsql/18/data/server.key
   sudo cp /etc/letsencrypt/live/postgres-rudder.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/root.crt
   ```

2. **Ajustement des permissions et propriétaires**

   ```bash
   sudo chown postgres:postgres /var/lib/pgsql/18/data/server.* /var/lib/pgsql/18/data/root.crt
   sudo chmod 600 /var/lib/pgsql/18/data/server.key
   sudo chmod 644 /var/lib/pgsql/18/data/server.crt /var/lib/pgsql/18/data/root.crt
   ```

3. **Activation de SSL dans `postgresql.conf`**
   Éditer le fichier `/var/lib/pgsql/18/data/postgresql.conf` :

   ```ini
   listen_addresses = '*'
   ssl = on
   ssl_cert_file = 'server.crt'
   ssl_key_file = 'server.key'
   ssl_ca_file = 'root.crt'
   ```

4. **Redémarrage de PostgreSQL et test de connexion TLS**

   ```bash
   sudo systemctl restart postgresql-18 && sudo systemctl status postgresql-18
   openssl s_client -connect postgres-rudder.abest.ovh:5432 -starttls postgres
   ```

---

## Étape 8 : Création de l'utilisateur et de la base de données

1. **Création de l'utilisateur et de la base Rudder**

   ```bash
   sudo -u postgres psql
   ```

   ```sql
   CREATE USER rudder WITH PASSWORD '123456789AD';
   CREATE DATABASE rudder OWNER rudder;
   GRANT ALL PRIVILEGES ON DATABASE rudder TO rudder;
   \q
   ```

2. **Configuration temporaire d'accès dans `pg_hba.conf`**
   Éditer `/var/lib/pgsql/18/data/pg_hba.conf` :

   ```text
   host    rudder          rudder          10.10.40.10/32          md5
   ```

3. **Rechargement de PostgreSQL**

   ```bash
   sudo systemctl restart postgresql-18
   ```

---

## Étape 9 : Migration de la base de données depuis l'ancien serveur (`plrudder`)

1. **Harmonisation du mot de passe sur le serveur source (`plrudder`)**
   Exécuter sur `plrudder` :

   ```bash
   sudo -u postgres psql -c "ALTER USER rudder WITH PASSWORD '123456789AD';"
   ```

2. **Exportation et import direct vers le nouveau serveur (`plbdd0`)**

   ```bash
   export PGPASSWORD="123456789AD"
   pg_dump -C -h localhost -U rudder rudder | psql -h plbdd0.ldap.abest.ovh -U rudder rudder
   ```

3. **Désactivation du service PostgreSQL local sur l'ancien serveur**
   Exécuter sur `plrudder` :

   ```bash
   sudo systemctl stop postgresql
   sudo systemctl disable postgresql
   sudo reboot
   ```

---

## Étape 10 : Sécurisation et chiffrement strict (`hostssl`)

1. **Mise à jour du fichier `pg_hba.conf` sur `plbdd0` pour forcer le SSL**
   Éditer `/var/lib/pgsql/18/data/pg_hba.conf` :

   ```text
   # TYPE    DATABASE        USER            ADDRESS                 METHOD
   hostssl   rudder          rudder          10.10.40.10/32          scram-sha-256
   hostssl   rudder          rudder          10.10.40.10/32          cert
   #host     rudder          rudder          10.10.40.10/32          scram-sha-256
   ```

2. **Redémarrage du service PostgreSQL**

   ```bash
   sudo systemctl restart postgresql-18
   ```

3. **Mise à jour de la chaîne de connexion sur l'application Rudder (`plrudder`)**
   Éditer `/opt/rudder/etc/rudder-web.properties` :

   ```properties
   rudder.jdbc.url=jdbc:postgresql://postgres-rudder.abest.ovh:5432/rudder?sslmode=require
   #rudder.jdbc.url=jdbc:postgresql://plbdd0.ldap.abest.ovh:5432/rudder
   ```

4. **Redémarrage du service Rudder Web**

   ```bash
   sudo systemctl restart rudder-server
   ```
