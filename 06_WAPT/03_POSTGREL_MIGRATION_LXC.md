# Configuration et Déploiement de PostgreSQL 18 pour WAPT

Ce document décrit la procédure pas à pas pour installer, configurer et vérifier l'instance PostgreSQL 18 dédiée au serveur WAPT sur Enterprise Linux 10 (RHEL / AlmaLinux / Rocky Linux).

## 📋 Prérequis

* Privilèges `sudo` ou accès `root`.
* Système Enterprise Linux 10 (EL10).
* Port `5432/tcp` ouvert sur le pare-feu.

---

## 🛠️ Procédure d'Installation et Configuration

### 1. Configuration du Firewall

Autoriser le port PostgreSQL par défaut dans `firewalld` pour permettre l'accès réseau distant :

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

### 2. Installation de PostgreSQL 18

Ajouter le dépôt PGDG (PostgreSQL Global Development Group), désactiver le module natif et installer le paquet serveur ainsi que les modules complémentaires :

```bash
# Ajout du dépôt officiel PGDG pour EL10
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-10.2-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Désactivation du module PostgreSQL fourni par défaut
sudo dnf -qy module disable postgresql

# Installation du serveur et des extensions contrib
sudo dnf install -y postgresql18-server postgresql18-contrib
```

### 3. Initialisation et Démarrage du Service

Initialiser le cluster de base de données puis démarrer/activer le service au démarrage du système :

```bash
# Initialisation de la base de données
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb

# Activation au démarrage et lancement immédiat du service
sudo systemctl enable --now postgresql-18
```

### 4. Vérification Initiale de l'Instance

Exécuter des requêtes d'inspection rapide pour valider la version et localiser les fichiers de configuration importants :

```bash
# Vérifier la version installée
sudo -u postgres psql -c "select version();"

# Afficher l'emplacement du fichier de configuration principal
sudo -u postgres psql -c "SHOW config_file;"

# Afficher l'emplacement du fichier de règles d'accès (HBA)
sudo -u postgres psql -c "SHOW hba_file;"
```

### 5. Édition des Fichiers de Configuration

#### A. Accès au Shell d'Administration (Optionnel)
```bash
sudo -u postgres psql
```

#### B. Configuration Réseau (`postgresql.conf`)
Éditez le fichier de configuration principal (ex: pour configurer `listen_addresses = '*'` afin de recevoir les connexions réseau du serveur WAPT) :

```bash
vi /var/lib/pgsql/18/data/postgresql.conf
```

#### C. Configuration de la Sécurité et Authentification (`pg_hba.conf`)
Configurez les règles d'accès réseau pour autoriser l'utilisateur WAPT ou les sous-réseaux nécessaires :

```bash
vi /var/lib/pgsql/18/data/pg_hba.conf
```

### 6. Redémarrage et Validation du Service

Appliquer les configurations modifiées en redémarrant le service et en vérifiant immédiatement son état :

```bash
sudo systemctl restart postgresql-18 && sudo systemctl status postgresql-18
```

> **Note :** Si vous devez réajuster des règles d'accès après test, modifiez à nouveau le fichier `pg_hba.conf` puis rechargez le service :
> ```bash
> vi /var/lib/pgsql/18/data/pg_hba.conf
> sudo systemctl reload postgresql-18
> ```

---

## 📌 Récapitulatif des Emplacements Clés

| Élément | Chemin / Commande |
| --- | --- |
| **Fichier de configuration principal** | `/var/lib/pgsql/18/data/postgresql.conf` |
| **Fichier de contrôle d'accès** | `/var/lib/pgsql/18/data/pg_hba.conf` |
| **Binaire d'initialisation** | `/usr/pgsql-18/bin/postgresql-18-setup` |
| **Nom du service systemd** | `postgresql-18` |
