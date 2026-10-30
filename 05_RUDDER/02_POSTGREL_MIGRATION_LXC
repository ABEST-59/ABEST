# Guide d'Installation de PostgreSQL 18 et Migration de Base de Données

Ce document récapitule la procédure d'installation et de configuration de PostgreSQL 18 sur un système basés sur Enterprise Linux (RHEL / AlmaLinux / Rocky Linux 10), ainsi que la migration d'une base de données via `pg_dump`.

---

## 📋 Prérequis

* Privilèges `sudo` ou accès `root`.
* Système RHEL/EL (v10 dans cet exemple) connecté à Internet.
* Port `5432/tcp` ouvert sur le pare-feu.

---

## 🛠️ Step-by-Step Procedure

### 1. Configuration du Pare-feu (Firewall)

Autoriser le port PostgreSQL par défaut (5432/tcp) pour permettre les connexions réseau :

```bash
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload
```

---

### 2. Installation de PostgreSQL 18

Ajouter le dépôt officiel PGDG (PostgreSQL Global Development Group), désactiver le module par défaut de la distribution et installer PostgreSQL 18 :

```bash
# Ajout du dépôt PGDG pour EL-10
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-10.2-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Désactivation du module PostgreSQL fourni par le système
sudo dnf -qy module disable postgresql

# Installation du serveur et des extensions contrib
sudo dnf install -y postgresql18-server postgresql18-contrib
```

---

### 3. Initialisation et Démarrage du Service

Initialiser la base de données cluster et activer le service au démarrage :

```bash
# Initialisation de la base de données
sudo /usr/pgsql-18/bin/postgresql-18-setup initdb

# Activation et démarrage du service
sudo systemctl enable --now postgresql-18
```

---

### 4. Configuration de PostgreSQL

Modifiez les fichiers de configuration selon vos besoins d'accès et d'écoute réseau.

#### A. Écoute réseau (`postgresql.conf`)
Éditez le fichier pour modifier l'adresse d'écoute (ex: `listen_addresses = '*'` ou IP spécifique) :
```bash
sudo vi /var/lib/pgsql/18/data/postgresql.conf
```

#### B. Contrôle d'accès par hôte (`pg_hba.conf`)
Configurez les règles d'authentification client (définir les adresses IP autorisées et les méthodes d'authentification comme `scram-sha-256` ou `md5`) :
```bash
sudo vi /var/lib/pgsql/18/data/pg_hba.conf
```

#### C. Aplliquer les modifications
Redémarrez le service pour prendre en compte les changements de configuration :
```bash
sudo systemctl restart postgresql-18
```

---

### 5. Migration / Copie de la Base de Données

Exemple de transfert direct de la base de données `rudder` du serveur local vers un serveur distant (`10.10.50.11`) :

```bash
# Définir le mot de passe dans la variable d'environnement (optionnel / temporaire)
export PGPASSWORD="VOTRE_MOT_DE_PASSE"

# Dump et restauration en direct (piped)
pg_dump -C -h localhost -U rudder rudder | psql -h 10.10.50.11 -U rudder rudder
```

---

### 6. Vérification et Administration

Se connecter à l'instance locale via l'utilisateur `postgres` ou directement sur la base concernée :

```bash
# Connexion en tant qu'administrateur postgres
sudo -u postgres psql

# Connexion directe à la base 'rudder'
sudo -u postgres psql -d rudder
```

---

## 📌 Commandes Utiles de Maintenance

| Action | Commande |
| :--- | :--- |
| Redémarrer PostgreSQL | `sudo systemctl restart postgresql-18` |
| Statut du service | `sudo systemctl status postgresql-18` |
| Emplacement des données | `/var/lib/pgsql/18/data/` |
