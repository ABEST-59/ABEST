# 📖 Guide Procédural : Installation et Configuration d'un Fleet Server (Elastic Stack)

---

## 📌 Contextualisation & Architecture

* **Cluster Elasticsearch :** 5 nœuds (`10.10.40.21` à `10.10.40.25`).
* **Nœud d'enrôlement initial (ES) :** `https://10.10.40.21:9200` (`plelk1`).
* **Empreinte de certificat CA (Fingerprint) :** `480b3ec345fdf932ac8d4db3ea7e7294b7837291589d7458ef751549621d485b`
* **Serveur Fleet Server :** Host dédié (`plfleet`).
* **Port d'écoute Fleet Server :** `8220/TCP`
* **Version Elastic Agent :** `9.5.5`

---

## 🛠️ Étape 1 : Configuration de la Haute Disponibilité sur Kibana

Puisque les sorties (*Outputs*) de Fleet sont verrouillées au niveau de la configuration de Kibana, l'ensemble des 5 nœuds du cluster doit être déclaré dans `/etc/kibana/kibana.yml`. Cela garantit le basculement automatique (*failover*) des agents en cas d'indisponibilité d'un nœud Elasticsearch.

1. Éditer le fichier de configuration Kibana :
   ```bash
   sudo nano /etc/kibana/kibana.yml
   ```

2. Configurer les blocs de sortie avec la liste complète des 5 nœuds :
   ```yaml
   # URL publique de Kibana pour la communication avec Fleet
   server.publicBaseUrl: "http://<IP_OU_DOMAINE_KIBANA>:5601"

   # Connexion native de Kibana à Elasticsearch
   elasticsearch.hosts: ["https://10.10.40.21:9200"]

   # Configuration de la sortie par défaut Fleet (Haute Disponibilité sur 5 nœuds)
   xpack.fleet.outputs:
     - id: fleet-default-output
       name: default
       is_default: true
       is_default_monitoring: true
       type: elasticsearch
       hosts:
         - "https://10.10.40.21:9200"
         - "https://10.10.40.22:9200"
         - "https://10.10.40.23:9200"
         - "https://10.10.40.24:9200"
         - "https://10.10.40.25:9200"
       ca_trusted_fingerprint: "480b3ec345fdf932ac8d4db3ea7e7294b7837291589d7458ef751549621d485b"
   ```

3. Redémarrer le service Kibana :
   ```bash
   sudo systemctl restart kibana
   ```

---

## 🎛️ Étape 2 : Génération du Service Token depuis Kibana

1. Ouvrir l'interface web Kibana.
2. Accéder à **Management** > **Fleet** > **Settings**.
3. S'assurer que le champ **Fleet Server hosts** contient bien l'URL de votre serveur Fleet (`https://<IP_PLFLEET>:8220`).
4. Dans l'onglet **Agents**, cliquer sur **Add Fleet Server**.
5. Récupérer le **Service Token** généré (champ `--fleet-server-service-token`).

---

## 🚀 Étape 3 : Installation de l'Elastic Agent en mode Fleet Server

> ⚠️ **Note d'installation (Linux / Hardening) :**  
> Si la partition `/var` ou `/var/lib` du serveur est montée avec l'option `noexec`, n'utilisez **pas** le paquet RPM (échec lors des scripts d'installation `%post`). Utilisez exclusivement l'archive tarball (`.tar.gz`) sous l'architecture exacte (`x86_64`).

### 1. Préparation de l'environnement
Sur le serveur hôte `plfleet` :

```bash
# Arrêter et supprimer toute installation précédente si nécessaire
sudo systemctl stop elastic-agent 2>/dev/null || true
sudo elastic-agent uninstall -f 2>/dev/null || true
sudo rpm -e elastic-agent 2>/dev/null || true

# Se placer dans /root
cd /root

# Télécharger l'archive Elastic Agent (Architecture Linux x86_64)
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-9.5.5-linux-x86_64.tar.gz

# Extraire l'archive et entrer dans le répertoire
tar xzvf elastic-agent-9.5.5-linux-x86_64.tar.gz
cd elastic-agent-9.5.5-linux-x86_64
```

### 2. Exécution de la commande d'enrôlement Fleet Server

Exécuter la commande d'installation en mode serveur :

```bash
./elastic-agent install \
  --fleet-server-es=https://10.10.40.21:9200 \
  --fleet-server-service-token=<VOTRE_SERVICE_TOKEN> \
  --fleet-server-policy=fleet-server-policy \
  --fleet-server-es-ca-trusted-fingerprint=480b3ec345fdf932ac8d4db3ea7e7294b7837291589d7458ef751549621d485b \
  --fleet-server-port=8220 \
  --install-servers
```

---

## 🔍 Étape 4 : Validation et Maintenance

### 1. Contrôle du service Linux (`systemd`)
Vérifier l'état de fonctionnement du service :
```bash
systemctl status elastic-agent
```

Pour consulter les journaux applicatifs en temps réel :
```bash
journalctl -u elastic-agent -f --no-pager
```

### 2. Validation dans Kibana
1. Naviguer vers Kibana > **Management** > **Fleet** > **Agents**.
2. Trouver l'agent correspondant au Fleet Server (`plfleet`).
3. Vérifier que son statut indique **Healthy** (Vert).

---

## 📌 Guide de dépannage (Troubleshooting)

| Symptôme / Message d'erreur | Cause probable | Action corrective |
| :--- | :--- | :--- |
| `Exec format error` | Téléchargement d'un binaire pour une mauvaise architecture CPU (ex: `arm64` au lieu de `x86_64`). | Vérifier l'architecture avec `uname -m` et télécharger la version `linux-x86_64`. |
| `Permission denied` / `exit status 126` | Option de montage `noexec` présente sur `/var` ou `/var/lib`. | Privilégier la méthode d'installation via tarball `.tar.gz` dans `/opt/Elastic/Agent`. |
| `no space left on device` | Espace disque suffisant manquant sur la partition cible (`/` ou `/opt`). | Nettoyer les paquets/caches (`dnf clean all`), supprimer les archives temporaires ou étendre le volume. |
| `Error: cannot perform installation as Elastic Agent is already running` | Instance Elastic Agent déjà active en cours d'exécution. | Arrêter le service (`systemctl stop elastic-agent`), purger l'instance (`elastic-agent uninstall -f`), puis relancer. |
| *Sortie gérée en dehors de Fleet* dans Kibana | Sortie verrouillée par le fichier de configuration Kibana. | Ajouter les nœuds directement dans le bloc `xpack.fleet.outputs` du fichier `/etc/kibana/kibana.yml`. |
