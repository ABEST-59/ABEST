# Documentation d'Installation du Serveur WAPT 2.6 (Enterprise / RedHat / RHEL)

Ce document récapitule la procédure pas à pas pour installer et configurer un serveur WAPT 2.6 sur une distribution basée sur RedHat/EL (Enterprise Linux) avec génération de certificat Let's Encrypt via DNS-OVH.

## 1. Prérequis et Configuration Système

### Configuration de la langue et de la locale

```bash
dnf install langpacks-en glibc-langpack-en -y
update-locale LANG=en_US.UTF-8
localectl
```

### Vérifications système

```bash
date
timedatectl status
sestatus
systemctl status firewalld
```

### Configuration du Pare-feu (Firewalld)

```bash
firewall-cmd --permanent --zone=internal --change-interface=ens18
firewall-cmd --permanent --zone=internal --add-port=443/tcp
firewall-cmd --reload
```

---

## 2. Installation des Dépendances de Base

```bash
# Installation des paquets système et EPEL
dnf install epel-release wget sudo -y

# Activation du module Nginx approprié
dnf module enable nginx:1.24 -y

# Installation des dépendances Kerberos et authentification Nginx
dnf install nginx-mod-http-auth-spnego krb5-workstation msktutil -y
```

---

## 3. Configuration du Certificat SSL Let's Encrypt (DNS OVH)

```bash
# Installation de Certbot et du plugin OVH
sudo dnf install -y certbot python3-certbot-dns-ovh

# Création du fichier d'identifiants OVH API
vi /root/ovh.ini
```

> **Exemple de contenu pour `/root/ovh.ini` :**
>
> ```ini
> dns_ovh_endpoint = ovh-eu
> dns_ovh_application_key = xxx
> dns_ovh_application_secret = xxx
> dns_ovh_consumer_key = xxx
> ```

```bash
# Sécurisation du fichier de clés
chmod 600 /root/ovh.ini

# Génération du certificat Let's Encrypt via challenge DNS
certbot certonly --dns-ovh --dns-ovh-credentials /root/ovh.ini -d wapt.abest.ovh
```

---

## 4. Ajout du Dépôt WAPT et Installation

### Configuration du Dépôt YUM/DNF

#### Pour RedHat 9 / EL9 :

```bash
cat > /etc/yum.repos.d/wapt.repo <<EOF
[wapt]
name=WAPT Server Repo
baseurl=https://wapt.tranquil.it/redhat9/wapt-2.6/
enabled=1
gpgcheck=1
EOF

wget -q -O /tmp/tranquil_it.gpg "https://wapt.tranquil.it/redhat9/RPM-GPG-KEY-TISWAPT-9"
rpm --import /tmp/tranquil_it.gpg
```

*(Optionnel) Pour RedHat 10 / EL10 :*

```bash
cat > /etc/yum.repos.d/wapt.repo <<EOF
[wapt]
name=WAPT Server Repo
baseurl=https://wapt.tranquil.it/redhat10/wapt-2.6/
enabled=1
gpgcheck=1
EOF

wget -q -O /tmp/tranquil_it.gpg "https://wapt.tranquil.it/redhat10/RPM-GPG-KEY-TISWAPT-10"
rpm --import /tmp/tranquil_it.gpg
```

### Installation du Serveur WAPT

```bash
dnf install tis-waptserver tis-waptsetup cabextract -y
```

---

## 5. Configuration de la Base de Données PostgreSQL

Éditez le fichier de configuration du serveur WAPT :

```bash
vi /opt/wapt/conf/waptserver.ini
```

Ajoutez ou modifiez le bloc de configuration de la base de données :

```ini
db_type = postgresql
db_host = postgres-wapt.abest.ovh
db_port = 5432
db_name = waptdb
db_user = wapt
db_password = 123456789AD
db_connect_timeout = 10
```

---

## 6. Post-Installation WAPT

Lancez le script de configuration interactif :

```bash
/opt/wapt/waptserver/scripts/postconf.sh
```

---

## 7. Copie / Liaison des Certificats pour WAPT

Créez les liens symboliques (ou copiez les fichiers) depuis le dossier Certbot vers l'emplacement attendu par le serveur WAPT / Nginx :

```bash
# Création du répertoire SSL s'il n'existe pas encore
mkdir -p /opt/wapt/waptserver/ssl/

# Liaison symbolique du certificat et de la clé privée
ln -sf /etc/letsencrypt/live/wapt.abest.ovh/fullchain.pem /opt/wapt/waptserver/ssl/cert.pem
ln -sf /etc/letsencrypt/live/wapt.abest.ovh/privkey.pem /opt/wapt/waptserver/ssl/key.pem

# Ajustement des permissions
chmod 644 /opt/wapt/waptserver/ssl/cert.pem
chmod 600 /opt/wapt/waptserver/ssl/key.pem
```