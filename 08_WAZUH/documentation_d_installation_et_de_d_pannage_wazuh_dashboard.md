# Guide d'Installation, Configuration et Dépannage - Wazuh Dashboard

Ce document récapitule la procédure complète d'installation, de configuration des certificats (internes et Let's Encrypt), de gestion des templates et de dépannage du service **Wazuh Dashboard**.

## 📋 Table des matières

1. [Ajout des dépôts et installation](#1-ajout-des-dépôts-et-installation)
2. [Génération du certificat Let's Encrypt (DNS OVH)](#2-génération-du-certificat-lets-encrypt-dns-ovh)
3. [Configuration des certificats internes (OpenSearch)](#3-configuration-des-certificats-internes-opensearch)
4. [Fichiers de configuration Wazuh Dashboard](#4-fichiers-de-configuration-wazuh-dashboard)
   * [4.1 opensearch_dashboards.yml (Avec Let's Encrypt)](#41-etcwazuh-dashboardopensearch_dashboardsyml-avec-lets-encrypt)
   * [4.2 wazuh.yml](#42-usrsharewazuh-dashboarddatawazuhconfigwazuhyml)
5. [Gestion de la Sécurité et Mots de Passe](#5-gestion-de-la-sécurité-et-mots-de-passe)
6. [Indexation et Ingestion des Templates Wazuh](#6-indexation-et-ingestion-des-templates-wazuh)
7. [Configuration du Pare-feu (firewalld)](#7-configuration-du-pare-feu-firewalld)
8. [Commandes de Diagnostic et Dépannage](#8-commandes-de-diagnostic-et-dépannage)

---

## 1. Ajout des dépôts et installation

Import de la clé GPG officielle Wazuh et ajout du dépôt sur une distribution basée sur RHEL/CentOS/AlmaLinux :

```bash
# Import de la clé GPG
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

# Configuration du dépôt yum / dnf
echo -e '[wazuh]\ngpgcheck=1\ngpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH\nenabled=1\nname=EL-$releasever - Wazuh\nbaseurl=https://packages.wazuh.com/4.x/yum/\npriority=1' | tee /etc/yum.repos.d/wazuh.repo

# Installation de Wazuh Dashboard
dnf -y install wazuh-dashboard
```

---

## 2. Génération du certificat Let's Encrypt (DNS OVH)

Génération non interactive du certificat SSL/TLS valide publiquement pour le nom de domaine `wazuh.abest.ovh` avec validation DNS via l'API OVH.

```bash
# Génération non interactive du certificat
sudo certbot certonly \
  --dns-ovh \
  --dns-ovh-credentials /root/ovh.ini \
  -d wazuh.abest.ovh \
  --non-interactive \
  --agree-tos \
  -m admin@abest.ovh
```

### Copie et droits pour Wazuh Dashboard
Pour autoriser Wazuh Dashboard à lire les certificats Let's Encrypt (ou en créant des liens/copies sécurisés) :

```bash
# Chemins d'origine Let's Encrypt :
# Certificat public : /etc/letsencrypt/live/wazuh.abest.ovh/fullchain.pem
# Clé privée :         /etc/letsencrypt/live/wazuh.abest.ovh/privkey.pem

# Attribution des droits de lecture à l'utilisateur wazuh-dashboard
chown -R root:wazuh-dashboard /etc/letsencrypt/archive/wazuh.abest.ovh
chown -R root:wazuh-dashboard /etc/letsencrypt/live/wazuh.abest.ovh
chmod 750 /etc/letsencrypt/live/wazuh.abest.ovh
chmod 750 /etc/letsencrypt/archive/wazuh.abest.ovh
chmod 640 /etc/letsencrypt/live/wazuh.abest.ovh/*
```

---

## 3. Configuration des certificats internes (OpenSearch)

Décompression des certificats internes pour la communication sécurisée entre le Dashboard et les nœuds Indexer OpenSearch (`10.10.40.31` à `10.10.40.35`).

```bash
NODE_NAME=plwdashboard

# Création du répertoire dédié
mkdir -p /etc/wazuh-dashboard/certs

# Extraction des certificats à partir de l'archive
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem

# Renommage standardisé des certificats
[ ! -e /etc/wazuh-dashboard/certs/dashboard.pem ] && mv -n /etc/wazuh-dashboard/certs/$NODE_NAME.pem /etc/wazuh-dashboard/certs/dashboard.pem
[ ! -e /etc/wazuh-dashboard/certs/dashboard-key.pem ] && mv -n /etc/wazuh-dashboard/certs/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem

# Application des permissions restrictives
chmod 500 /etc/wazuh-dashboard/certs
chmod 400 /etc/wazuh-dashboard/certs/*
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

---

## 4. Fichiers de configuration Wazuh Dashboard

### 4.1 `/etc/wazuh-dashboard/opensearch_dashboards.yml` (Avec Let's Encrypt)

Dans cette configuration :
* **`server.ssl.certificate`** et **`server.ssl.key`** pointent vers le certificat **Let's Encrypt** (pour l'accès Web sécurisé des utilisateurs).
* **`opensearch.ssl.certificateAuthorities`** conserve le **`root-ca.pem`** interne pour la liaison sécurisée avec l'API OpenSearch Indexer.

```yaml
server.host: 10.10.40.40
server.port: 443
opensearch.hosts: ["https://10.10.40.31:9200","https://10.10.40.32:9200","https://10.10.40.33:9200","https://10.10.40.34:9200","https://10.10.40.35:9200"]

# Authentification et autorisations
opensearch.ssl.verificationMode: certificate
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: ["kibana_read_only"]

# Certificat HTTPS Serveur Web (Let's Encrypt)
server.ssl.enabled: true
server.ssl.certificate: "/etc/letsencrypt/live/wazuh.abest.ovh/fullchain.pem"
server.ssl.key: "/etc/letsencrypt/live/wazuh.abest.ovh/privkey.pem"

# Certificat CA pour la communication interne vers OpenSearch
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]

# Redirection par défaut et sessions
uiSettings.overrides.defaultRoute: /app/wz-home
opensearch_security.cookie.ttl: 900000
opensearch_security.session.ttl: 900000
opensearch_security.session.keepalive: true
```

### 4.2 `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml`

Configuration du plugin Wazuh pour communiquer avec l'API du serveur Wazuh Manager (`10.10.40.30:55000`) :

```yaml
hosts:
  - default:
      url: https://10.10.40.30
      port: 55000
      username: wazuh-wui
      password: KL.aB9x2kw20wa6i0J77W1NFk8BImCqP
      run_as: true
```

### Application et Redémarrage du Service

```bash
systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl restart wazuh-dashboard
```

---

## 5. Gestion de la Sécurité et Mots de Passe

Stockage sécurisé des mots de passe dans le Keystore OpenSearch Dashboards :

```bash
# Ajout du mot de passe OpenSearch dans le keystore
echo "C5hGOSutX6kqqqtbd4Tsz?*qb8.74qtK" | /usr/share/wazuh-dashboard/bin/opensearch-dashboards-keystore --allow-root add -f --stdin opensearch.password

# Désactivation du dépôt après installation
sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/wazuh.repo
```

---

## 6. Indexation et Ingestion des Templates Wazuh

Téléchargement et application du modèle d'index Wazuh v4.7.4 dans OpenSearch Indexer :

```bash
cd /tmp/

# Téléchargement du template officiel Wazuh v4.7.4
curl -so /tmp/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.7.4/extensions/elasticsearch/7.x/wazuh-template.json

# Envoi du template vers l'Indexer OpenSearch
curl -k -u "admin:g+Ijpxz8K9g9r?CsFp..zD0vf1UAxdWG" -X PUT "https://10.10.40.31:9200/_template/wazuh" \
  -H "Content-Type: application/json" \
  -d @/tmp/wazuh-template.json
```

---

## 7. Configuration du Pare-feu (firewalld)

Autorisation du port HTTPS (443) :

```bash
# Activation de firewalld
sudo dnf install -y firewalld && sudo systemctl enable --now firewalld

# Ouverture du port HTTPS 443
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

---

## 8. Commandes de Diagnostic et Dépannage

### Diagnostic et Logs

```bash
# Suivi en direct des logs du Dashboard
journalctl -u wazuh-dashboard -f

# Statut du service
systemctl status wazuh-dashboard
```

### Test d'accès aux certificats Let's Encrypt par le Dashboard

```bash
sudo -u wazuh-dashboard cat /etc/letsencrypt/live/wazuh.abest.ovh/fullchain.pem >/dev/null && echo "Let's Encrypt Cert OK"
sudo -u wazuh-dashboard cat /etc/letsencrypt/live/wazuh.abest.ovh/privkey.pem >/dev/null && echo "Let's Encrypt Key OK"
```