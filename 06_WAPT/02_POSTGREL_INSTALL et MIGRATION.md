# 🗄️ Procédure d'Installation & Configuration PostgreSQL 18 (Serveur PLBDD4)

Cette procédure détaille l'installation, la sécurisation SSL (Certbot / OVH), la configuration réseau et l'initialisation de la base de données WAPT sur un serveur RHEL / Rocky Linux 9.

---

## 📋 Prérequis

* Serveur RHEL 9 / Rocky Linux 9 connecté au réseau.
* Accès `sudo` ou privilèges `root`.
* Clés d'API OVH configurées pour la validation DNS Let's Encrypt.
* Flux réseau autorisés vers la machine cliente (`10.10.40.12/32`).

---

## 🚀 1. Installation de PostgreSQL 18

1. **Ajouter le dépôt PGDG (PostgreSQL Global Development Group) :**
   ```bash
   sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   ```

2. **Désactiver le module PostgreSQL natif RHEL :**
   ```bash
   sudo dnf -qy module disable postgresql
   ```

3. **Installer PostgreSQL 18 et les outils contrib :**
   ```bash
   sudo dnf install -y postgresql18-server postgresql18-contrib
   ```

4. **Initialiser l'instance de base de données :**
   ```bash
   sudo /usr/pgsql-18/bin/postgresql-18-setup initdb
   ```

5. **Activer et démarrer le service :**
   ```bash
   sudo systemctl enable --now postgresql-18
   ```

6. **Vérifier l'installation et l'emplacement des fichiers :**
   ```bash
   sudo -u postgres psql -c "select version();"
   sudo -u postgres psql -c "SHOW config_file;"
   sudo -u postgres psql -c "SHOW hba_file;"
   ```

---

## 🛡️ 2. Configuration du Pare-feu (Firewalld)

1. **Assigner l'interface à la zone `internal` et ouvrir le port 5432 :**
   ```bash
   sudo firewall-cmd --permanent --zone=internal --change-interface=ens18
   sudo firewall-cmd --permanent --zone=internal --add-port=5432/tcp
   sudo firewall-cmd --reload
   ```

---

## 🔐 3. Génération & Application des Certificats SSL (Certbot + OVH)

1. **Installer Certbot et le plugin DNS OVH :**
   ```bash
   sudo dnf install -y certbot python3-certbot-dns-ovh
   ```

2. **Créer le fichier de clés d'API OVH :**
   ```bash
   sudo vi /root/ovh.ini
   sudo chmod 600 /root/ovh.ini
   ```

3. **Générer le certificat SSL via le challenge DNS OVH :**
   ```bash
   sudo certbot certonly --dns-ovh --dns-ovh-credentials /root/ovh.ini -d postgres-wapt.abest.ovh
   ```

4. **Copier les certificats dans le répertoire de données PostgreSQL :**
   ```bash
   sudo cp /etc/letsencrypt/live/postgres-wapt.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/server.crt
   sudo cp /etc/letsencrypt/live/postgres-wapt.abest.ovh/privkey.pem /var/lib/pgsql/18/data/server.key
   sudo cp /etc/letsencrypt/live/postgres-wapt.abest.ovh/fullchain.pem /var/lib/pgsql/18/data/root.crt
   ```

5. **Ajuster la propriété et les permissions des certificats :**
   ```bash
   sudo chown postgres:postgres /var/lib/pgsql/18/data/server.* /var/lib/pgsql/18/data/root.crt
   sudo chmod 600 /var/lib/pgsql/18/data/server.key
   sudo chmod 644 /var/lib/pgsql/18/data/server.crt /var/lib/pgsql/18/data/root.crt
   ```

---

## ⚙️ 4. Configuration de PostgreSQL

### A. Écoute réseau (`postgresql.conf`)
Éditez le fichier `/var/lib/pgsql/18/data/postgresql.conf` :
```ini
listen_addresses = '*'

# Configuration SSL (À activer une fois le certificat validé)
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
```

### B. Contrôle d'accès (`pg_hba.conf`)
Éditez le fichier `/var/lib/pgsql/18/data/pg_hba.conf` pour autoriser le serveur WAPT :
```text
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    waptdb          wapt            10.10.40.14/32          scram-sha-256
```

### C. Redémarrer le service
```bash
sudo systemctl restart postgresql-18 && sudo systemctl status postgresql-18
```

---

## 👤 5. Création de la Base de Données & Utilisateur WAPT

1. **Ouvrir l'invite PostgreSQL :**
   ```bash
   sudo -u postgres psql
   ```

2. **Exécuter les requêtes SQL de création :**
   ```sql
   CREATE USER wapt WITH PASSWORD '123456789AD';
   CREATE DATABASE waptdb OWNER wapt;
   GRANT ALL PRIVILEGES ON DATABASE waptdb TO wapt;
   \q
   ```

---

## 🧪 6. Test de Connexion SSL

Vérifiez que le serveur accepte correctement les connexions chiffrées SSL / TLS :
```bash
openssl s_client -connect postgres-bastion.abest.ovh:5432 -starttls postgres
```
