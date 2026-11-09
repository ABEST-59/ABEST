# Guide d'Installation, Configuration et Dépannage - Wazuh Dashboard

Ce document récapitule la procédure complète d'installation, de configuration des certificats, de gestion des templates et de dépannage du service **Wazuh Dashboard**.

## 📋 Table des matières

1. [Ajout des dépôts et installation](#1-ajout-des-dépôts-et-installation)
2. [Configuration des certificats SSL/TLS](#2-configuration-des-certificats-ssltls)
3. [Fichiers de configuration Wazuh Dashboard](#3-fichiers-de-configuration-wazuh-dashboard)
   - [3.1 opensearch_dashboards.yml](#31-etcwazuh-dashboardopensearch_dashboardsyml)
   - [3.2 wazuh.yml](#32-usrsharewazuh-dashboarddatawazuhconfigwazuhyml)
4. [Gestion de la Sécurité et Mots de Passe](#4-gestion-de-la-sécurité-et-mots-de-passe)
5. [Indexation et Ingestion des Templates Wazuh](#5-indexation-et-ingestion-des-templates-wazuh)
6. [Configuration du Pare-feu (firewalld)](#6-configuration-du-pare-feu-firewalld)
7. [Commandes de Diagnostic et Vérification](#7-commandes-de-diagnostic-et-vérification)

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

## 2. Configuration des certificats SSL/TLS

Décompression et mise en place des certificats pour la communication sécurisée entre le Dashboard et les nœuds Indexer.

```bash
NODE_NAME=plwdashboard

# Création du répertoire dédié
mkdir -p /etc/wazuh-dashboard/certs

# Extraction des certificats à partir de l'archive d'installation
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem

# Renommage standardisé des certificats
[ ! -e /etc/wazuh-dashboard/certs/dashboard.pem ] && mv -n /etc/wazuh-dashboard/certs/$NODE_NAME.pem /etc/wazuh-dashboard/certs/dashboard.pem
[ ! -e /etc/wazuh-dashboard/certs/dashboard-key.pem ] && mv -n /etc/wazuh-dashboard/certs/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem

# Application des permissions restrictives
chmod 500 /etc/wazuh-dashboard/certs
chmod 400 /etc/wazuh-dashboard/certs/*
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs
```

---

## 3. Fichiers de configuration Wazuh Dashboard

### 3.1 `/etc/wazuh-dashboard/opensearch_dashboards.yml`

Configuration principale de liaison du Dashboard avec la grappe d'indexeurs OpenSearch (`10.10.40.31` à `10.10.40.35`) et écoute sur l'adresse IP `10.10.40.40` sur le port HTTPS 443 :

```yaml
server.host: 10.10.40.40
server.port: 443
opensearch.hosts: ["https://10.10.40.31:9200","https://10.10.40.32:9200","https://10.10.40.33:9200","https://10.10.40.34:9200","https://10.10.40.35:9200"]
#opensearch.username: kibanaserver
#opensearch.password: C5hGOSutX6kqqqtbd4Tsz?*qb8.74qtK
opensearch.ssl.verificationMode: certificate
opensearch.requestHeadersAllowlist: ["securitytenant","Authorization"]
opensearch_security.multitenancy.enabled: false
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
server.ssl.enabled: true
server.ssl.key: "/etc/wazuh-dashboard/certs/dashboard-key.pem"
server.ssl.certificate: "/etc/wazuh-dashboard/certs/dashboard.pem"
opensearch.ssl.certificateAuthorities: ["/etc/wazuh-dashboard/certs/root-ca.pem"]
uiSettings.overrides.defaultRoute: /app/wz-home
# Session expiration settings
opensearch_security.cookie.ttl: 900000
opensearch_security.session.ttl: 900000
opensearch_security.session.keepalive: true
```

### 3.2 `/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml`

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

### Prise en compte des modifications

```bash
systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl restart wazuh-dashboard
```

---

## 4. Gestion de la Sécurité et Mots de Passe

Stockage sécurisé des mots de passe dans le Keystore OpenSearch Dashboards :

```bash
# Ajout du mot de passe OpenSearch dans le keystore
echo "C5hGOSutX6kqqqtbd4Tsz?*qb8.74qtK" | /usr/share/wazuh-dashboard/bin/opensearch-dashboards-keystore --allow-root add -f --stdin opensearch.password

# Désactivation du dépôt après installation pour éviter des mises à jour accidentelles
sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/wazuh.repo
```

---

## 5. Indexation et Ingestion des Templates Wazuh

Téléchargement et application du modèle d'index (template) Wazuh v4.7.4 dans OpenSearch Indexer :

```bash
cd /tmp/

# Téléchargement du template officiel Wazuh v4.7.4
curl -so /tmp/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.7.4/extensions/elasticsearch/7.x/wazuh-template.json

# Envoi du template vers l'Indexer OpenSearch (exemple sur l'IP 10.10.40.31)
curl -k -u "admin:g+Ijpxz8K9g9r?CsFp..zD0vf1UAxdWG" -X PUT "https://10.10.40.31:9200/_template/wazuh" \
  -H "Content-Type: application/json" \
  -d @/tmp/wazuh-template.json
```

---

## 6. Configuration du Pare-feu (firewalld)

Autorisation du port HTTPS (443) pour permettre l'accès web au Dashboard :

```bash
# Installation et activation du pare-feu
sudo dnf install -y firewalld && sudo systemctl enable --now firewalld

# Ouverture du port HTTPS 443
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

---

## 7. Commandes de Diagnostic et Vérification

### Inspection des services et des journaux

```bash
# Suivi en direct des logs du Dashboard
journalctl -u wazuh-dashboard -f

# État du service
systemctl status wazuh-dashboard
systemctl restart wazuh-dashboard

# Test direct en mode verbeux (pour débogage avancé)
sudo -u wazuh-dashboard /usr/share/wazuh-dashboard/bin/opensearch-dashboards --allow-root --verbose
```

### Vérification de l'état du cluster et des nœuds Indexer

```bash
# Test de connectivité à l'API Indexer
curl -sk -u admin:'2S9TEpzUX.3Ww4eSvDxvF*l.GC0kjPgY' https://10.10.40.32:9200

# Santé du cluster
curl -sk -u admin:'2S9TEpzUX.3Ww4eSvDxvF*l.GC0kjPgY' https://10.10.40.32:9200/_cluster/health?pretty

# Liste des nœuds et des index
curl -sk -u admin:'2S9TEpzUX.3Ww4eSvDxvF*l.GC0kjPgY' https://10.10.40.32:9200/_cat/nodes?v
curl -sk -u admin:'2S9TEpzUX.3Ww4eSvDxvF*l.GC0kjPgY' https://10.10.40.32:9200/_cat/indices?v

# Vérification rapide sur plusieurs nœuds du cluster
for ip in 31 32 33 34 35; do
  echo "===== 10.10.40.$ip ====="
  curl -sk -u admin:'2S9TEpzUX.3Ww4eSvDxvF*l.GC0kjPgY' https://10.10.40.$ip:9200 | grep '"number"'
done
```

### Validation des Certificats et des Droits

```bash
# Vérification d'ouverture des ports locaux
ss -tulpn | grep :443

# Test de lecture des certificats par l'utilisateur du service
sudo -u wazuh-dashboard cat /etc/wazuh-dashboard/certs/root-ca.pem >/dev/null && echo "CA OK"
sudo -u wazuh-dashboard cat /etc/wazuh-dashboard/certs/dashboard.pem >/dev/null && echo "Cert OK"
sudo -u wazuh-dashboard cat /etc/wazuh-dashboard/certs/dashboard-key.pem >/dev/null && echo "Key OK"

# Validation SSL via OpenSSL
openssl s_client -connect 10.10.40.32:9200 -CAfile /etc/wazuh-dashboard/certs/root-ca.pem
```
