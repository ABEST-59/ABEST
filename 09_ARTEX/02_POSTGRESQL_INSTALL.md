# Installation et configuration de PostgreSQL 18

Ce document résume les étapes d'installation, d'initialisation et de configuration de **PostgreSQL 18** sur un système de type RHEL / Enterprise Linux (EL), incluant l'ouverture du pare-feu, la création d'un utilisateur et d'une base de données dédiée (`artexdb`), ainsi que l'autorisation de connexion distante.

## 1. Ajout du dépôt officiel PostgreSQL et installation

1. **Installer le dépôt PGDG (PostgreSQL Global Development Group)** :

   ```
   sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-10.2-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   
   ```

2. **Désactiver le module PostgreSQL intégré par défaut** afin d'éviter les conflits de versions :

   ```
   sudo dnf -qy module disable postgresql
   
   ```

3. **Installer le serveur PostgreSQL 18 et les modules additionnels (`contrib`)** :

   ```
   sudo dnf install -y postgresql18-server postgresql18-contrib
   
   ```

## 2. Initialisation et démarrage du service

1. **Initialiser le cluster de bases de données** :

   ```
   sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
   
   ```

2. **Activer le démarrage automatique et démarrer le service** :

   ```
   sudo systemctl enable --now postgresql-18
   
   ```

## 3. Configuration du pare-feu (`firewalld`)

1. **Installer `firewalld`** (si non présent) :

   ```
   sudo dnf install -y firewalld
   
   ```

2. **Activer et démarrer `firewalld`** :

   ```
   sudo systemctl enable --now firewalld
   
   ```

3. **Autoriser le port PostgreSQL (5432/tcp) de façon permanente et recharger les règles** :

   ```
   sudo firewall-cmd --add-port=5432/tcp --permanent
   sudo firewall-cmd --reload
   
   ```

> *Note : Lors de vos commandes précédentes, une tentative incomplète (`--reloa`) avait été saisie avant la commande valide `--reload`.*

## 4. Création de l'utilisateur et de la base de données

> ⚠️ **Sécurité** : Remplacez le mot de passe d'exemple (`123456789AD`) par un mot de passe fort en environnement de production, et évitez de le laisser en clair dans l'historique shell.

1. **Créer l'utilisateur `artex`** :

   ```
   su - postgres -c "psql -c \"CREATE USER artex WITH PASSWORD '123456789AD';\""
   
   ```

   *(Une ligne précédente incomplète sans fermeture de guillemets avait été ignorée/interrompue avant cette commande valide.)*

2. **Créer la base de données `artexdb` appartenant à `artex`** :

   ```
   su - postgres -c "psql -c \"CREATE DATABASE artexdb OWNER artex;\""
   
   ```

## 5. Configuration réseau et authentification

Éditez les fichiers de configuration principaux du cluster PostgreSQL 18 :

1. **Paramètres du serveur (`postgresql.conf`)** — autoriser l'écoute sur toutes les interfaces réseau :

   ```
   sudo vi /var/lib/pgsql/18/data/postgresql.conf
   
   ```

   Ajoutez ou modifiez la directive suivante :

   ```
   listen_addresses = '*'
   
   ```

2. **Règles d'authentification client (`pg_hba.conf`)** — autoriser l'utilisateur `artex` à se connecter à `artexdb` depuis l'hôte `10.10.40.14` avec `scram-sha-256` :

   ```
   sudo vi /var/lib/pgsql/18/data/pg_hba.conf
   
   ```

   Ajoutez la règle suivante :

   ```
   host    artexdb         artex           10.10.40.14/32          scram-sha-256
   
   ```

## 6. Redémarrage du système

Pour appliquer l'ensemble des changements système/réseau ou vérifier la bonne reprise des services après redémarrage :

```
sudo reboot

```

*(Optionnel : après le redémarrage, vous pouvez vérifier l'état du service avec `sudo systemctl status postgresql-18`.)*
