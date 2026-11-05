# Procédure d'installation et de configuration du Bastion Teleport

Ce document récapitule les étapes d'installation et de configuration pour le serveur Bastion Teleport (`bastion.abest.ovh`), la configuration des enregistrements DNS, l'obtention des certificats SSL/TLS Wildcard via OVH DNS, l'installation de l'agent WAPT, la configuration du pare-feu ainsi que la jonction au domaine Active Directory / LDAP.

## 1. Configuration des enregistrements DNS (Active Directory / Windows DNS)

Avant ou pendant le déploiement de Teleport, configurez la zone DNS de votre serveur (ex: `pldc1.ldap.abest.ovh`) sous le sous-domaine `bastion.abest.ovh` afin de router l'ensemble des sous-domaines applicatifs Web de Teleport vers l'adresse IP du serveur Bastion (`10.10.40.12`).

### Enregistrements à créer dans la zone `abest.ovh` > `bastion` :

| Nom / Hôte | Type | Données / Adresse IP | Description |
| ----- | ----- | ----- | ----- |
| **(identique au dossier parent)** | Hôte (A) | `10.10.40.12` | Redirige `bastion.abest.ovh` vers le serveur Teleport |
| **`*`** | Hôte (A) | `10.10.40.12` | Enregistrement Wildcard pour router tous les sous-domaines (`*.bastion.abest.ovh`) |

## 2. Prérequis et Dépendances

Activation du dépôt EPEL et installation des outils nécessaires pour Certbot (avec le plugin DNS OVH).

```bash
sudo dnf install -y epel-release
sudo dnf install -y certbot python3-certbot-dns-ovh
```

## 3. Génération des certificats SSL/TLS (Let's Encrypt + OVH)

### 3.1 Configuration des identifiants OVH API

Créez le fichier de clés API OVH et appliquez les permissions restreintes.

```bash
sudo vi /root/ovh.ini
sudo chmod 600 /root/ovh.ini
```

> **Structure recommandée pour `/root/ovh.ini` :**
>
> ```ini
> dns_ovh_endpoint = ovh-eu
> dns_ovh_application_key = <votre_application_key>
> dns_ovh_application_secret = <votre_application_secret>
> dns_ovh_consumer_key = <votre_consumer_key>
> ```

### 3.2 Génération du certificat Wildcard

Demandez un certificat pour le domaine principal ainsi que l'ensemble du sous-domaine (`*.bastion.abest.ovh`) :

```bash
sudo certbot certonly \
  --dns-ovh \
  --dns-ovh-credentials /root/ovh.ini \
  -d bastion.abest.ovh \
  -d '*.bastion.abest.ovh'
```

*Note : En cas de soucis d'exécution ou de propagation DNS, consultez les logs de certbot :*

```bash
tail -f /var/log/letsencrypt/letsencrypt.log
```

## 4. Installation et Configuration de Teleport

### 4.1 Installation des paquets

```bash
sudo dnf install -y https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm
```

### 4.2 Configuration de Teleport

Créez le dossier de configuration et le fichier `/etc/teleport.yaml` :

```bash
sudo mkdir -p /etc/teleport
sudo vi /etc/teleport.yaml
```

**Exemple de fichier `/etc/teleport.yaml` :**

```yaml
version: v3
teleport:
  nodename: plbastion
  data_dir: /var/lib/teleport
  join_params:
    token_name: 2a76a81241eefd2292396e8b9c0ebfd9
    method: token
  log:
    output: /var/log/teleport.log
    severity: INFO
    format:
      output: text
  ca_pin: ""
  diag_addr: ""
  # Stockage de l'état du cluster dans PostgreSQL
  storage:
    type: postgres
    conn_string: "postgres://teleport:<MOT_DE_PASSE>@postgres-bastion.abest.ovh:5432/teleportdb?sslmode=verify-full"

auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  cluster_name: bastion.abest.ovh
  proxy_listener_mode: multiplex

ssh_service:
  enabled: "yes"
  labels:
    env: PRD

proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: bastion.abest.ovh:443
  https_keypairs:
    - cert_file: /etc/letsencrypt/live/bastion.abest.ovh/fullchain.pem
      key_file: /etc/letsencrypt/live/bastion.abest.ovh/privkey.pem
```

### 4.3 Démarrage et activation du service

```bash
# Vérifier la présence de l'utilisateur dédié (si nécessaire)
id teleport

# Démarrer et activer le service au démarrage
sudo systemctl enable --now teleport
sudo systemctl status teleport

# Suivre les logs de fonctionnement
tail -f /var/log/teleport.log
```

## 5. Configuration du Pare-feu (Firewalld)

Installation, activation du service Firewalld et ouverture des flux HTTP/HTTPS requis.

```bash
# Installation et démarrage
sudo dnf install -y firewalld
sudo systemctl enable --now firewalld

# Ouverture des ports
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

## 6. Déploiement du client WAPT

Installation de l'agent WAPT, application de la configuration centrale et attribution d'un identifiant unique (UUID).

```bash
# Installation de l'agent WAPT
sudo dnf install -y https://wapt.abest.ovh/wapt/waptagent/tis-waptagent.x86_64.rpm

# Configuration et réinitialisation à partir de l'URL centrale
sudo wapt-get reset-config-from-url https://wapt.abest.ovh/wapt/conf.d/default_1f1479ed483a8ca354f0886c44dccd0524156a8bc4141e559eec8744bd474991.json
sudo wapt-get generate-uuid

# Redémarrage du service WAPT
sudo systemctl restart waptservice
```

## 7. Jonction au Domaine AD / LDAP (Realmd)

Jonction du serveur dans l'Unité d'Organisation (OU) ciblée pour les serveurs Linux.

```bash
echo '<MOT_DE_PASSE>' | sudo realm join ldap.abest.ovh \
  -U user.ldap \
  --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=DSI,OU=ABEST,DC=ldap,DC=abest,DC=ovh" \
  --stdin-password
```

## 8. Maintenance et Commandes Utiles

* **Redémarrer Teleport :** `sudo systemctl restart teleport`
* **Vérifier les logs Teleport :** `journalctl -fu teleport` ou `tail -f /var/log/teleport.log`
* **Vérifier le statut du domaine AD :** `realm list`
