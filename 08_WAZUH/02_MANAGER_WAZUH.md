# Installation et Configuration du Wazuh Manager & Filebeat (`plwmanager`)

Ce document détaille la procédure complète de déploiement et d'intégration du **Wazuh Manager** (`10.10.40.41`) connecté à un cluster de **5 nœuds Wazuh Indexer** (`10.10.40.31` à `10.10.40.35`).

---

## 📋 Prérequis et Architecture

- **Machine / LXC** : `plwmanager` (`10.10.40.41`)
- **OS** : Rocky Linux 10
- **Dépôt certifié** : Wazuh 4.x
- **Certificats SSL** : Préalablement générés (`wazuh-certificates.tar`)

---

## 🛠️ 1. Installation des Paquets (Manager & Filebeat)

Exécutez les commandes suivantes sur `plwmanager` :

```bash
# Importation de la clé GPG et ajout du dépôt Wazuh
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
echo -e '[wazuh]\ngpgcheck=1\ngpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH\nenabled=1\nname=EL-$releasever - Wazuh\nbaseurl=https://packages.wazuh.com/4.x/yum/\npriority=1' | tee /etc/yum.repos.d/wazuh.repo

# Installation de Wazuh Manager, Filebeat et outils requises
dnf -y install wazuh-manager filebeat tar
```

---

## 🔐 2. Placement des Certificats SSL pour Filebeat

Déployez les certificats générés pour le Manager :

```bash
NODE_NAME=plwmanager

mkdir -p /etc/filebeat/certs
tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem

# Renommage des fichiers au format standard Filebeat
mv -n /etc/filebeat/certs/$NODE_NAME.pem /etc/filebeat/certs/filebeat.pem
mv -n /etc/filebeat/certs/$NODE_NAME-key.pem /etc/filebeat/certs/filebeat-key.pem

# Permissions strictes sur les certificats
chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs
```

---

## ⚙️ 3. Configuration de Filebeat

### 3.1 Téléchargement des templates et modules Wazuh

```bash
# Fichier de configuration modèle
curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.14/tpl/wazuh/filebeat/filebeat.yml

# Template d'indexation Wazuh pour Elasticsearch / OpenSearch
curl -so /etc/filebeat/wazuh-template.json https://raw.githubusercontent.com/wazuh/wazuh/v4.14.10/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json

# Module Filebeat propre à Wazuh
curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz | tar -xvz -C /usr/share/filebeat/module
```

### 3.2 Gestion des identifiants (Keystore Filebeat)

```bash
filebeat keystore create
echo 'admin' | filebeat keystore add username --stdin --force
echo 'admin' | filebeat keystore add password --stdin --force
```

### 3.3 Fichier `/etc/filebeat/filebeat.yml` (Equilibrage de charge 5 nœuds)

```yaml
# Wazuh - Filebeat configuration file

output.elasticsearch:
  hosts:
    - "10.10.40.31:9200"
    - "10.10.40.32:9200"
    - "10.10.40.33:9200"
    - "10.10.40.34:9200"
    - "10.10.40.35:9200"
  loadbalance: true
  protocol: https
  username: "${username}"
  password: "${password}"
  ssl.certificate_authorities:
    - /etc/filebeat/certs/root-ca.pem
  ssl.certificate: "/etc/filebeat/certs/filebeat.pem"
  ssl.key: "/etc/filebeat/certs/filebeat-key.pem"
  ssl.verification_mode: "none"

setup.template.json.enabled: true
setup.template.json.path: '/etc/filebeat/wazuh-template.json'
setup.template.json.name: 'wazuh'
setup.ilm.overwrite: true
setup.ilm.enabled: false

filebeat.modules:
  - module: wazuh
    alerts:
      enabled: true
    archives:
      enabled: false

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644

logging.metrics.enabled: false

seccomp:
  default_action: allow
  syscalls:
  - action: allow
    names:
    - rseq
```

---

## 🔑 4. Configuration du Keystore Wazuh Manager

Pour permettre au processus natif de Wazuh Manager d'interagir directement avec l'API du cluster d'indexeurs, configurez son trousseau de clés :

```bash
# Si le trousseau de clés rencontre un blocage ou est corrompu :
systemctl stop wazuh-manager
rm -f /var/ossec/queue/keystore

# Injection des identifiants dans le trousseau de clés du Manager
echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k username
echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k password
```

---

## 📄 5. Configuration de `ossec.conf` (`/var/ossec/etc/ossec.conf`)

Mettez à jour le bloc `<indexer>` dans le fichier de configuration du Manager pour communiquer avec l'ensemble des 5 nœuds :

```xml
  <!-- Integration avec le cluster Wazuh Indexer (5 noeuds) -->
  <indexer>
    <enabled>yes</enabled>
    <hosts>
      <host>https://10.10.40.31:9200</host>
      <host>https://10.10.40.32:9200</host>
      <host>https://10.10.40.33:9200</host>
      <host>https://10.10.40.34:9200</host>
      <host>https://10.10.40.35:9200</host>
    </hosts>
    <ssl>
      <certificate_authorities>
        <ca>/etc/filebeat/certs/root-ca.pem</ca>
      </certificate_authorities>
      <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
      <key>/etc/filebeat/certs/filebeat-key.pem</key>
    </ssl>
  </indexer>
```

Appliquez les permissions et droits appropriés :
```bash
chown root:wazuh /var/ossec/etc/ossec.conf
chmod 640 /var/ossec/etc/ossec.conf
```

---

## 🚀 6. Démarrage des Services & Validation

```
0. **Firewalld) :
   ```bash
   sudo firewall-cmd --permanent --add-port=1514/tcp && sudo firewall-cmd --permanent --add-port=1515/tcp && sudo firewall-cmd --reload
   ```

```
1. **Désactiver le dépôt Wazuh** (bonne pratique pour éviter les mises à jour inattendues) :
   ```bash
   sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/wazuh.repo
   ```

2. **Démarrer les services et recharger Systemd** :
   ```bash
   systemctl daemon-reload
   systemctl enable --now wazuh-manager
   systemctl enable --now filebeat
   ```

3. **Tester l'envoi des journaux Filebeat** :
   ```bash
   filebeat test output
   ```

4. **Vérifier le statut du Manager et du Cluster** :
   ```bash
   systemctl status wazuh-manager.service
   /var/ossec/bin/cluster_control -l
   ```
