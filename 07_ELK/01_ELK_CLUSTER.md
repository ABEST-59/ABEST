# Elasticsearch 9.5.5 - Air-Gapped Production Cluster Guide

Guide complet de déploiement, de configuration et d'exploitation d'un cluster Elasticsearch hautement disponible à **5 nœuds** segmenté par rôles sur **Rocky Linux 9**, dans un environnement isolé (**air-gapped**).

---

## 🏗️ 1. Architecture & Topologie du Cluster

Le cluster se nomme **`cluster-elk`** et répartit les charges entre des nœuds de pilotage (Master/Ingest) et des nœuds de stockage (Data).

| Nom du Nœud | Adresse IP | Rôles principaux | Description |
| :--- | :--- | :--- | :--- |
| **`plelk1`** | `10.10.40.21` | `master`, `ingest`, `transform`, `ml` | Nœud maître principal & API REST |
| **`plelk2`** | `10.10.40.22` | `master`, `ingest`, `transform`, `ml` | Nœud maître secondaire & Ingestion |
| **`plelk3`** | `10.10.40.23` | `master`, `data_hot`, `data_warm`, `data_cold` | Nœud de données |
| **`plelk4`** | `10.10.40.24` | `master`, `data_hot`, `data_warm`, `data_cold` | Nœud de données |
| **`plelk5`** | `10.10.40.25` | `master`, `data_hot`, `data_warm`, `data_cold` | Nœud de données |

* **Stockage dédié :**
  * Répertoire de données : `/ELK/data`
  * Répertoire des logs : `/ELK/logs`
  * Répertoire temporaire JVM : `/ELK/tmp`

---

## 🚀 2. Déploiement & Configuration Initiale (Pas à Pas)

### Étape 1 : Installation du paquet RPM
```bash
sudo dnf install https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-9.5.5-x86_64.rpm -y
```

### Étape 2 : Préparation des répertoires et permissions
```bash
sudo mkdir -p /ELK/data /ELK/logs /ELK/tmp
sudo chown -R elasticsearch:elasticsearch /ELK
```

### Étape 3 : Tuning JVM (`/etc/elasticsearch/jvm.options.d/`)
Configuration de l'emplacement temporaire, de la Heap Memory et de la rotation des logs GC :

```bash
# Redirection du dossier TMP
echo "-Djava.io.tmpdir=/ELK/tmp" > /etc/elasticsearch/jvm.options.d/tmp.options

# Définition de la taille du Heap (ex: 512M)
cat << 'EOF' > /etc/elasticsearch/jvm.options.d/heap.options
-Xms512m
-Xmx512m
EOF

# Journalisation GC (Garbage Collector) avec rotation de fichiers
cat << 'EOF' > /etc/elasticsearch/jvm.options.d/gc.options
-Xlog:gc*,gc+age=trace,safepoint:file=/ELK/logs/gc.log:utctime,level,pid,tags:filecount=32,filesize=64m
EOF

# Ajustement des permissions
chown -R elasticsearch:elasticsearch /etc/elasticsearch/jvm.options.d/
```

---

## 🔐 3. Enrôlement des Nœuds Nouveaux/Secondaires

Pour joindre un nœud secondaire (ex: `plelk2` à `plelk5`) au cluster initié par `plelk1`, utilisez la commande de réconfiguration par jeton :

```bash
/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <VOTRE_TOKEN_ENROLLMENT>
```

---

## ⚙️ 4. Configurations Types (`/etc/elasticsearch/elasticsearch.yml`)

### A. Configuration pour un nœud Master / Ingest (`plelk1`)
```yaml
cluster.name: cluster-elk
node.name: plelk1
node.roles: ["master", "ingest", "transform", "ml", "remote_cluster_client"]

path.data: /ELK/data
path.logs: /ELK/logs

network.host: 10.10.40.21
http.port: 9200
transport.port: 9300

discovery.seed_hosts: ["10.10.40.21", "10.10.40.22", "10.10.40.23", "10.10.40.24", "10.10.40.25"]
cluster.initial_master_nodes: ["plelk1", "plelk2", "plelk3", "plelk4", "plelk5"]

xpack.security.enabled: true
xpack.security.enrollment.enabled: true

xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12

http.host: 0.0.0.0
transport.host: 0.0.0.0
```

### B. Configuration pour un nœud Data (`plelk3`)
*(Modifier `node.name` et `network.host` selon l'IP du nœud)*
```yaml
cluster.name: cluster-elk
node.name: plelk3
node.roles: ["master", "data_hot", "data_warm", "data_cold"]

path.data: /ELK/data
path.logs: /ELK/logs

network.host: 10.10.40.23
http.port: 9200
transport.port: 9300

discovery.seed_hosts: ["10.10.40.21", "10.10.40.22", "10.10.40.23", "10.10.40.24", "10.10.40.25"]
cluster.initial_master_nodes: ["plelk1", "plelk2", "plelk3", "plelk4", "plelk5"]

xpack.security.enabled: true
xpack.security.enrollment.enabled: true

xpack.security.http.ssl:
  enabled: true
  keystore.path: certs/http.p12

xpack.security.transport.ssl:
  enabled: true
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12

http.host: 0.0.0.0
transport.host: 0.0.0.0
```

---

## 🔒 5. Configuration du Pare-Feu (`firewalld`)

Configuration réseau basée sur la zone d'interface interne (`internal`) pour sécuriser la carte `eth0` :

```bash
# Installation et activation du pare-feu
dnf install -y firewalld
systemctl enable --now firewalld

# Affectation de l'interface et ouverture des ports REST & Transport
firewall-cmd --permanent --zone=internal --change-interface=eth0
firewall-cmd --permanent --zone=internal --add-port=9200/tcp
firewall-cmd --permanent --zone=internal --add-port=9300/tcp

# Rechargement des règles
firewall-cmd --reload
```

---

## 🛠️ 6. Procédures d'Exploitation & Dépannage

### A. Réinitialisation des données pour conflit d'UUID
Si un nœud secondaire refuse de joindre le cluster avec une erreur `CoordinationStateRejectedException` :

```bash
systemctl stop elasticsearch
rm -rf /ELK/data/*
systemctl start elasticsearch
```

### B. Réaffectation de rôles de nœud (`repurpose`)
Pour nettoyer les fragments de shards inutilisés lors d'une modification des rôles d'un nœud dans `elasticsearch.yml` :

```bash
systemctl stop elasticsearch
/usr/share/elasticsearch/bin/elasticsearch-node repurpose
systemctl start elasticsearch
```

### C. Inspection des logs & diagnostics
```bash
# Suivi en direct des journaux du cluster
tail -f /ELK/logs/cluster-elk.log

# Analyse des erreurs système du service
journalctl -xeu elasticsearch.service
```

---

## 🧪 7. Validation et Commandes de Test

À exécuter depuis un nœud principal (`plelk1`) :

* **Santé du cluster :**
  ```bash
  curl -k -u elastic:<password> https://10.10.40.21:9200/_cluster/health?pretty
  ```

* **Vérification des 5 nœuds connectés :**
  ```bash
  curl -k -u elastic:<password> https://10.10.40.21:9200/_cat/nodes?v
  ```

* **Vérification de l'allocation des shards :**
  ```bash
  curl -k -u elastic:<password> https://10.10.40.21:9200/_cat/shards?v
