# Déploiement d'un Cluster Wazuh Indexer à 5 Nœuds sur LXC (Rocky Linux 10)

Ce document détaille la procédure complète d'installation, de configuration et d'optimisation pour le déploiement d'un cluster hautement disponible **Wazuh Indexer** (basé sur OpenSearch) sur conteneurs LXC Rocky Linux 10, optimisé pour des environnements à ressources contraintes (1 Go RAM / 2 vCPU par nœud).

## 📐 Architecture du Cluster

| Hôte / LXC | Adresse IP | Rôles OpenSearch | Description | 
| ----- | ----- | ----- | ----- | 
| **`plwazuh1`** | `10.10.40.31` | `cluster_manager` | Nœud Maître 1 | 
| **`plwazuh2`** | `10.10.40.32` | `cluster_manager` | Nœud Maître 2 | 
| **`plwazuh3`** | `10.10.40.33` | `data`, `ingest` | Nœud de Données 1 | 
| **`plwazuh4`** | `10.10.40.34` | `data`, `ingest` | Nœud de Données 2 | 
| **`plwazuh5`** | `10.10.40.35` | `data`, `ingest` | Nœud de Données 3 | 
| **`plwmanager`** | `10.10.40.30` | N/A | Serveur Wazuh Manager | 
| **`plwdashboard`** | `10.10.40.40` | N/A | Interface d'administration | 

---

## 📜 1. Fichiers de Configuration

### 1.1 `config.yml` (Génération des Certificats SSL)

Ce fichier est utilisé sur la machine centrale pour générer les certificats de l'ensemble du cluster.

```yaml
nodes:
  indexer:
    - name: plwazuh1
      ip: "10.10.40.31"
      dn: "CN=plwazuh1,OU=Wazuh,O=Wazuh,L=California,C=US"
    - name: plwazuh2
      ip: "10.10.40.32"
      dn: "CN=plwazuh2,OU=Wazuh,O=Wazuh,L=California,C=US"
    - name: plwazuh3
      ip: "10.10.40.33"
      dn: "CN=plwazuh3,OU=Wazuh,O=Wazuh,L=California,C=US"
    - name: plwazuh4
      ip: "10.10.40.34"
      dn: "CN=plwazuh4,OU=Wazuh,O=Wazuh,L=California,C=US"
    - name: plwazuh5
      ip: "10.10.40.35"
      dn: "CN=plwazuh5,OU=Wazuh,O=Wazuh,L=California,C=US"
  server:
    - name: plwmanager
      ip: "10.10.40.41"
      dn: "CN=plwmanager,OU=Wazuh,O=Wazuh,L=California,C=US"
  dashboard:
    - name: plwdashboard
      ip: "10.10.40.40"
      dn: "CN=plwdashboard,OU=Wazuh,O=Wazuh,L=California,C=US"
```

### 1.2 `opensearch.yml` (Nœuds Maîtres : `plwazuh1` & `plwazuh2`)

Fichier `/etc/wazuh-indexer/opensearch.yml` pour **`plwazuh1`** (Adapter `node.name` et `network.host` pour `plwazuh2`) :

```yaml
network.host: "10.10.40.31"
http.port: 9200
transport.port: 9300
node.name: "plwazuh1"
node.roles: ["cluster_manager"]

cluster.name: "wazuh-cluster"
cluster.initial_cluster_manager_nodes: ["plwazuh1", "plwazuh2"]
discovery.seed_hosts: ["10.10.40.31", "10.10.40.32", "10.10.40.33", "10.10.40.34", "10.10.40.35"]

path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer

plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.resolve_hostname: false

plugins.security.authcz.admin_dn:
- "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
- "CN=plwazuh1,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh2,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh3,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh4,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh5,OU=Wazuh,O=Wazuh,L=California,C=US"

plugins.security.restapi.roles_enabled:
- "all_access"
- "security_rest_api_access"

plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".plugins-ml-model", ".plugins-ml-task", ".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opensearch-notifications-*", ".opensearch-notebooks", ".opensearch-observability", ".opendistro-asynchronous-search-response*", ".replication-metadata-store"]

### Option to allow Filebeat-oss 7.10.2 to work ###
compatibility.override_main_response_version: true
```

### 1.3 `opensearch.yml` (Nœuds de Données : `plwazuh3`, `plwazuh4` & `plwazuh5`)

Fichier `/etc/wazuh-indexer/opensearch.yml` pour **`plwazuh3`** (Adapter `node.name` et `network.host` pour `plwazuh4` et `plwazuh5`) :

```yaml
network.host: "10.10.40.33"
http.port: 9200
transport.port: 9300
node.name: "plwazuh3"
node.roles: ["data", "ingest"]

cluster.name: "wazuh-cluster"
cluster.initial_cluster_manager_nodes: ["plwazuh1", "plwazuh2"]
discovery.seed_hosts: ["10.10.40.31", "10.10.40.32", "10.10.40.33", "10.10.40.34", "10.10.40.35"]

path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer

plugins.security.ssl.http.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.http.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.http.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: /etc/wazuh-indexer/certs/indexer.pem
plugins.security.ssl.transport.pemkey_filepath: /etc/wazuh-indexer/certs/indexer-key.pem
plugins.security.ssl.transport.pemtrustedcas_filepath: /etc/wazuh-indexer/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: false
plugins.security.ssl.transport.resolve_hostname: false

plugins.security.authcz.admin_dn:
- "CN=admin,OU=Wazuh,O=Wazuh,L=California,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
- "CN=plwazuh1,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh2,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh3,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh4,OU=Wazuh,O=Wazuh,L=California,C=US"
- "CN=plwazuh5,OU=Wazuh,O=Wazuh,L=California,C=US"

plugins.security.restapi.roles_enabled:
- "all_access"
- "security_rest_api_access"

plugins.security.system_indices.enabled: true
plugins.security.system_indices.indices: [".plugins-ml-model", ".plugins-ml-task", ".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opensearch-notifications-*", ".opensearch-notebooks", ".opensearch-observability", ".opendistro-asynchronous-search-response*", ".replication-metadata-store"]

### Option to allow Filebeat-oss 7.10.2 to work ###
compatibility.override_main_response_version: true
```

---

## 🛠️ 2. Guide d'Installation Pas à Pas

### Étape 1 : Préparation du système et Génération des Certificats

Sur votre machine d'administration :

```bash
# Téléchargement des outils de génération
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml

# Éditer config.yml avec la structure des 5 nœuds (voir section 1.1)
vi config.yml

# Générer les certificats
bash ./wazuh-certs-tool.sh -A

# Paqueter l'archive pour déploiement
tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
rm -rf ./wazuh-certificates
```

### Étape 2 : Prérequis système sur l'hôte physique LXC

1. Le paramètre `vm.max_map_count` **doit être appliqué sur l'hôte physique** hébergeant les conteneurs LXC :
   ```bash
   sysctl -w vm.max_map_count=262144
   echo "vm.max_map_count=262144" >> /etc/sysctl.conf
   ```

2. **Système de fichiers et permissions `/var`** :
   La partition `/var` de vos conteneurs (ou de l'hôte) **ne doit pas** être montée avec les options restrictive `noexec`, `nosuid` ou `nodev`. Java et la bibliothèque JNA doivent pouvoir exécuter des binaires natifs extraits sous `/var/lib/wazuh-indexer`.
   ```bash
   # Vérifier les options du point de montage sur l'hôte/LXC
   mount | grep " /var "
   ```
   *(Si les options `noexec`, `nosuid` ou `nodev` sont présentes, retirez-les de la ligne correspondant à `/var` dans le fichier `/etc/fstab`).*

### Étape 3 : Installation du Wazuh Indexer sur chaque nœud

Exécutez cette procédure sur **chacun des 5 nœuds conteneurs LXC** (`plwazuh1` à `plwazuh5`) :

```bash
# Installation des utilitaires et importation de la clé GPG Wazuh
dnf install -y coreutils
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH

# Ajout du dépôt officiel Wazuh
echo -e '[wazuh]\ngpgcheck=1\ngpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH\nenabled=1\nname=EL-$releasever - Wazuh\nbaseurl=https://packages.wazuh.com/4.x/yum/\npriority=1' | tee /etc/yum.repos.d/wazuh.repo

# Installation du paquet
dnf -y install wazuh-indexer

# Désactivation temporaire du dépôt post-installation (bonnes pratiques)
sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/wazuh.repo
```

### Étape 4 : Placement des Certificats SSL

Sur chaque nœud, définissez la variable `$NODE_NAME` (ex: `NODE_NAME=plwazuh1`) et extrayez les certificats :

```bash
export NODE_NAME=plwazuh1  # Adapter selon le nœud (plwazuh1..5)

# Création du dossier et extraction
mkdir -p /etc/wazuh-indexer/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem

# Renommage standardisé
mv -n /etc/wazuh-indexer/certs/$NODE_NAME.pem /etc/wazuh-indexer/certs/indexer.pem
mv -n /etc/wazuh-indexer/certs/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem

# Application des permissions strictes
chmod 500 /etc/wazuh-indexer/certs
chmod 400 /etc/wazuh-indexer/certs/*
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs

# Nettoyage
rm -f ./wazuh-certificates.tar
```

### Étape 5 : Optimisation mémoire JVM (RAM 1 Go / Swap 1 Go)

En raison de la contrainte de **1 Go de RAM**, réduisez la taille du Tas Java (Heap) et configurez un répertoire temporaire exécutable pour éviter les erreurs `com.sun.jna.Native` :

```bash
# Ajustement du Heap JVM à 300Mo pour libérer de la mémoire système
echo -e "-Xms300m\n-Xmx300m" > /etc/wazuh-indexer/jvm.options.d/heap.options

# Redirection du dossier temporaire JNA/Java dans /var (sans noexec)
mkdir -p /var/lib/wazuh-indexer/tmp
chown -R wazuh-indexer:wazuh-indexer /var/lib/wazuh-indexer/tmp
chmod 750 /var/lib/wazuh-indexer/tmp
echo "-Djava.io.tmpdir=/var/lib/wazuh-indexer/tmp" > /etc/wazuh-indexer/jvm.options.d/tmpdir.options
```

### Étape 6 : Démarrage du Service et Initialisation du Cluster

#### 1. Démarrer le service sur les 5 nœuds :

```bash
systemctl daemon-reload
systemctl enable --now wazuh-indexer.service
```

#### 2. Initialiser la sécurité (À exécuter UNIQUEMENT sur `plwazuh1`) :

Une fois que les 5 nœuds sont démarrés et connectés au réseau :

```bash
/usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

---

## 🔍 3. Vérification et Diagnostic

Pour valider l'état du cluster et vérifier que les 5 nœuds communiquent correctement :

### Test de réponse de l'API Indexer

```bash
[root@plwazuh1 ~]# curl -k -u admin https://10.10.40.31:9200
Enter host password for user 'admin':
```

*Résultat attendu :*

```json
{
  "name" : "plwazuh1",
  "cluster_name" : "wazuh-cluster",
  "cluster_uuid" : "YkPT6RFfTpS-YV-7Ts-lmQ",
  "version" : {
    "number" : "7.10.2",
    "build_type" : "rpm",
    "build_hash" : "b24f1114bfcde4db0350b4785bdc9c826b9aea40",
    "build_date" : "2026-11-04T11:28:30.795800892Z",
    "build_snapshot" : false,
    "lucene_version" : "9.12.3",
    "minimum_wire_compatibility_version" : "7.10.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
}
```

### Liste des nœuds et rôles actifs dans le cluster

```bash
curl -k -u admin:admin https://10.10.40.31:9200/_cat/nodes?v
```

*Résultat attendu :*

```text
ip          heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
10.10.40.31           35          82   2    0.15    0.10     0.05 m         *      plwazuh1
10.10.40.32           33          80   1    0.12    0.08     0.04 m         -      plwazuh2
10.10.40.33           40          85   3    0.20    0.15     0.08 di        -      plwazuh3
10.10.40.34           38          84   2    0.18    0.14     0.07 di        -      plwazuh4
10.10.40.35           39          83   2    0.17    0.13     0.06 di        -      plwazuh5
```
