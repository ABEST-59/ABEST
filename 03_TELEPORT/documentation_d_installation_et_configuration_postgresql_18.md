# Procedures d'Installation et de Configuration - PostgreSQL 18

Ce document retrace les étapes de déploiement et de configuration de **PostgreSQL 18** sur un système Enterprise Linux (EL), incluant la gestion des accès Active Directory / LDAP, la génération de certificats SSL avec Certbot (OVH DNS), et la sécurisation du service.

---

## 1. Installation du Dépôt PGDG et de PostgreSQL 18

Ajout du dépôt officiel PostgreSQL, désactivation du module PostgreSQL par défaut du système, puis installation du serveur et de ses outils.

```bash
# Ajout du dépôt PostgreSQL PGDG
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-10.2-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Désactivation du module par défaut du système
sudo dnf -qy module disable postgresql

# Installation des paquets PostgreSQL 18 et des extensions de base
sudo dnf install -y postgresql18-server postgresql18-contrib wal2json_18
```

---

## 2. Initialisation et Démarrage du Service

```bash
# Initialisation de la base de données
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb

# Démarrage et activation au démarrage du système
sudo systemctl enable --now postgresql-18

# Configuration du pare-feu (Ouverture du port 5432)
sudo firewall-cmd --add-port=5432/tcp --permanent
sudo firewall-cmd --reload
```

---

## 3. Configuration des Accès Administrateurs (Realm / Sudoers)

Autorisation du groupe LDAP / Active Directory à se connecter au serveur et à exécuter les commandes en `sudo`.

```bash
# Autoriser le groupe AD à se connecter via SSH
sudo realm permit -g SSHSRVADM

# Donner les privilèges Sudo au groupe AD
echo "%SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL" | sudo tee /etc/sudoers.d/domain_admins
```

---

## 4. Génération du Certificat SSL via Certbot (DNS OVH)

```bash
# Installation du dépôt EPEL et des paquets Certbot
sudo dnf install -y epel-release
sudo dnf install -y certbot python3-certbot-dns-ovh

# Configuration des identifiants OVH (Insérer les clés API dans le fichier)
sudo vi /root/ovh.ini
sudo chmod 600 /root/ovh.ini

# Génération du certificat SSL en mode non-interactif
sudo certbot certonly \
  --dns-ovh \
  --dns-ovh-credentials /root/ovh.ini \
  -d postgres-bastion.abest.ovh \
  --email s.thorez@outlook.fr \
  --agree-tos \
  --no-eff-email \
  --non-interactive
```

---

## 5. Configuration SSL pour PostgreSQL

Copie et sécurisation des certificats Let's Encrypt dans le répertoire de données de PostgreSQL.

```bash
# Copie des certificats dans le répertoire Data
sudo cp /etc/letsencrypt/live/postgres-bastion.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/server.crt
sudo cp /etc/letsencrypt/live/postgres-bastion.abest.ovh/privkey.pem /var/lib/pgsql/18/data/server.key
sudo cp /etc/letsencrypt/live/postgres-bastion.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/root.crt

# Application des permissions requises par PostgreSQL
sudo chown postgres:postgres /var/lib/pgsql/18/data/server.* /var/lib/pgsql/18/data/root.crt
sudo chmod 600 /var/lib/pgsql/18/data/server.key
sudo chmod 644 /var/lib/pgsql/18/data/server.crt /var/lib/pgsql/18/data/root.crt
```

---

## 6. Édition de la Configuration et Redémarrage

```bash
# Configuration des accès réseau et règles d'authentification
sudo vi /var/lib/pgsql/18/data/pg_hba.conf

# Redémarrage et vérification du statut du service
sudo systemctl restart postgresql-18 && sudo systemctl status postgresql-18
```