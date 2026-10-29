# Documentation : Déploiement et Configuration de Heartbeat Elastic (9.5.4)

Cette documentation décrit la procédure complète d'installation, de configuration et d'intégration du magasin de confiance SSL/TLS pour le service de supervision **Heartbeat** sur la sonde `plelkannexe` (`10.10.40.29`).

---

## 📋 Table des matières
1. [Vue d'ensemble](#1-vue-densemble)
2. [Préréquis & Services surveillés](#2-préréquis--services-surveillés)
3. [Installation & Configuration de Heartbeat](#4-installation--configuration-de-heartbeat)
4. [Fichier de Configuration (`/etc/heartbeat/heartbeat.yml`)](#5-fichier-de-configuration-etcheartbeatheartbeatyml)
5. [Validation, Initialisation et Démarrage](#6-validation-initialisation-et-démarrage)
6. [Visualisation dans Kibana](#7-visualisation-dans-kibana)

---

## 1. Vue d'ensemble

Heartbeat est un agent d'Elastic Stack exécutant du *Uptime Monitoring* (disponibilité HTTP/HTTPS, TCP, ICMP) et assurant le suivi en temps réel de la validité et des dates d'expiration des certificats SSL/TLS de l'infrastructure.

---

## 2. Préréquis & Services surveillés

* **Sonde de supervision :** `plelkannexe` (`10.10.40.29`) — RedHat / Rocky Linux.
* **Cluster Elastic / Kibana :** `https://10.10.40.21:5601`

### Périmètre des 6 services supervisés :

| Service | Hostname / URL | Verification SSL | Remarques |
| :--- | :--- | :--- | :--- |
| **OPNsense** | `https://opnsense.abest.ovh/` | `full` | Certificat Let's Encrypt (RSA) |
| **Rudder** | `https://rudder.abest.ovh/rudder/` | `full` | App. Java sur `/rudder/` + Certificat ECDSA (`Root X2`) |
| **WAPT** | `https://wapt.abest.ovh/` | `full` | Certificat Let's Encrypt (RSA) |
| **Kibana** | `https://kibana.abest.ovh:5601/` | `full` | Interface d'administration |
| **Proxmox VE** | `https://pve.abest.ovh:8006/` | `full` | Hyperviseur |
| **Bastion** | `https://bastion.abest.ovh/` | `full` | Passerelle d'accès |

---

## 3. Installation & Configuration de Heartbeat

Procédure pas-à-pas exécutée sur la machine `plelkannexe` :

```bash
# 1. Installation du package RPM Heartbeat 9.5.4
dnf install -y https://artifacts.elastic.co/downloads/beats/heartbeat/heartbeat-9.5.4-x86_64.rpm

# 2. Test de résolution DNS (optionnel)
dig artifacts.elastic.co +short

# 3. Création du keystore sécurisé pour les secrets
sudo heartbeat keystore create

# 4. Ajout du mot de passe Elasticsearch (ES_PASSWORD)
sudo heartbeat keystore add ES_PASSWORD

# 5. Sauvegarde de l'ancienne configuration si nécessaire
mv /etc/heartbeat/heartbeat.yml /etc/heartbeat/heartbeat.old1
```

---

## 4. Fichier de Configuration (`/etc/heartbeat/heartbeat.yml`)

Contenu final du fichier `/etc/heartbeat/heartbeat.yml` :

```yaml
################### Heartbeat Configuration #########################

heartbeat.config.monitors:
  path: ${path.config}/monitors.d/*.yml
  reload.enabled: false
  reload.period: 5s

# ============================= Moniteurs =====================================
heartbeat.monitors:
- type: http
  id: opnsense-monitor
  name: "OPNsense"
  enabled: true
  urls: ["https://opnsense.abest.ovh/"]
  schedule: '@every 60s'
  ssl.verification_mode: "full"

- type: http
  id: rudder-monitor
  name: "Rudder"
  enabled: true
  urls: ["https://rudder.abest.ovh/rudder/"]
  schedule: '@every 60s'
  ssl.verification_mode: "full"
  check.response.status: [200, 301, 302]

- type: http
  id: wapt-monitor
  name: "WAPT"
  enabled: true
  urls: ["https://wapt.abest.ovh/"]
  schedule: '@every 60s'
  ssl.verification_mode: "full"

- type: http
  id: kibana-monitor
  name: "Kibana"
  enabled: true
  urls: ["https://kibana.abest.ovh:5601/"]
  schedule: '@every 60s'
  ssl.verification_mode: "full"

- type: http
  id: pve-monitor
  name: "Proxmox VE"
  enabled: true
  urls: ["https://pve.abest.ovh:8006/"]
  schedule: '@every 60s'
  ssl.verification_mode: "full"

- type: http
  id: bastion-monitor
  name: "Bastion"
  enabled: true
  urls: ["https://bastion.abest.ovh/"]
  schedule: '@every 60s'
  ssl.verification_mode: "full"

# ======================= Elasticsearch template setting =======================
setup.template.settings:
  index.number_of_shards: 1
  index.codec: best_compression

# =================================== Kibana ===================================
setup.kibana:
  host: "https://10.10.40.21:5601"
  username: "elastic"
  password: "${ES_PASSWORD}"
  ssl.verification_mode: "none"

# =================================== Outputs ===================================
output.elasticsearch:
  hosts: [
    "https://10.10.40.21:9200",
    "https://10.10.40.22:9200",
    "https://10.10.40.23:9200",
    "https://10.10.40.24:9200",
    "https://10.10.40.25:9200"
  ]
  preset: balanced
  protocol: "https"
  username: "elastic"
  password: "${ES_PASSWORD}"
  ssl.verification_mode: "none"

# ================================= Processors =================================
processors:
  - add_observer_metadata:

# ================================== Logging ===================================
logging.level: info
```

---

## 5. Validation, Initialisation et Démarrage

Exécuter les commandes suivantes pour vérifier la configuration, charger les dashboards dans Kibana et activer le service :

```bash
# 1. Verification de la syntaxe du fichier heartbeat.yml
sudo heartbeat test config -e

# 2. Chargement des templates d'index et des dashboards Kibana
sudo heartbeat setup -e

# 3. Activation au démarrage et lancement du service
sudo systemctl enable --now heartbeat-elastic
```

---

## 6. Visualisation dans Kibana

1. Ouvrir l'interface Kibana : `https://10.10.40.21:5601`
2. Aller dans le menu **Observability** > **Uptime**.
3. **Onglet Monitors :** Vérifier que les 6 services affichent le statut **Opérationnel** (Vert).
4. **Onglet Certificates :** Contrôler le tableau de suivi des certificats SSL/TLS pour s'assurer que la durée d'expiration de chaque site est bien calculée à 90 jours (3 mois).
