# 📘 Documentation d'Installation et Configuration de Kibana avec Let's Encrypt (DNS OVH)

Cette documentation décrit la procédure pas à pas pour installer **Kibana** sur un système Enterprise Linux (RHEL, AlmaLinux, Rocky Linux, Fedora), l'enrôler sur un cluster Elasticsearch et sécuriser son accès via HTTPS grâce à un certificat **Let's Encrypt** généré automatiquement avec le challenge DNS OVH.

---

## 📌 Prérequis

* Un serveur Linux basé sur RHEL / AlmaLinux / Rocky Linux / Fedora (`plkibana`).
* Un nom de domaine configuré chez OVH (ex: `kibana.abest.ovh`).
* Un cluster Elasticsearch déjà fonctionnel et son jeton d'enrôlement (*enrollment token*).
* Privilèges d'accès `root` ou `sudo`.

---

## 🚀 1. Installation de Kibana

Téléchargez et installez le paquet RPM officiel de Kibana (version 9.5.5) :

```bash
sudo dnf install https://artifacts.elastic.co/downloads/kibana/kibana-9.5.5-x86_64.rpm -y
```

---

## 🔑 2. Enrôlement auprès d'Elasticsearch

Liez Kibana au cluster Elasticsearch à l'aide du jeton d'enrôlement généré préalablement sur l'un des nœuds Elasticsearch :

```bash
/usr/share/kibana/bin/kibana-setup --enrollment-token <VOTRE_TOKEN_ENROLLEMENT>
```

---

## 🔐 3. Génération des clés de chiffrement (Optionnel / Bonnes Pratiques)

Les clés de chiffrement évitent que les utilisateurs soient déconnectés à chaque redémarrage et permettent de chiffrer les objets sauvegardés (règles d'alerte, connecteurs, rapports PDF).

Si vous n'avez pas exécuté cette étape lors de la configuration initiale, vous pouvez générer ces clés manuellement :

```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate
```

> **À quoi servent ces clés ?**
> * `xpack.encryptedSavedObjects.encryptionKey` : Chiffre les identifiants et alertes stockés.
> * `xpack.security.encryptionKey` : Sécurise la session utilisateur (évite la déconnexion après un reboot).
> * `xpack.reporting.encryptionKey` : Sécurise la génération et l'exportation des rapports (PDF, CSV).

Les clés générées sont à copier à la fin du fichier `/etc/kibana/kibana.yml`.

---

## 🔒 4. Génération du certificat Let's Encrypt via DNS OVH

### 4.1. Installation de Certbot et du plugin OVH

Activez le dépôt EPEL puis installez Certbot ainsi que le module d'authentification DNS pour OVH :

```bash
sudo dnf install epel-release -y
sudo dnf install -y certbot python3-certbot-dns-ovh
```

### 4.2. Configuration des identifiants API OVH

Créez le fichier d'accès API OVH :

```bash
sudo vi /root/ovh.ini
```

Ajoutez vos identifiants API OVH :

```ini
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = xxxxxx
dns_ovh_application_secret = xxxxxx
dns_ovh_consumer_key = xxxxxx
```

Sécurisez les permissions du fichier :

```bash
sudo chmod 600 /root/ovh.ini
```

### 4.3. Obtention du certificat SSL/TLS

Générez le certificat pour votre sous-domaine via le challenge DNS :

```bash
sudo certbot certonly --dns-ovh --dns-ovh-credentials /root/ovh.ini -d kibana.abest.ovh
```

> ℹ️ Les certificats sont générés dans le répertoire `/etc/letsencrypt/live/kibana.abest.ovh/`.

---

## 📂 5. Copie et sécurisation des certificats pour Kibana

L'utilisateur système `kibana` n'a pas accès au dossier `/etc/letsencrypt/`. Copiez les certificats dans un dossier dédié `/etc/kibana/certs/` et appliquez les permissions restreintes :

```bash
sudo mkdir -p /etc/kibana/certs
sudo cp /etc/letsencrypt/live/kibana.abest.ovh/fullchain.pem /etc/kibana/certs/
sudo cp /etc/letsencrypt/live/kibana.abest.ovh/privkey.pem /etc/kibana/certs/

sudo chown -R kibana:kibana /etc/kibana/certs
sudo chmod 600 /etc/kibana/certs/privkey.pem
sudo chmod 644 /etc/kibana/certs/fullchain.pem
```

---

## ⚙️ 6. Configuration finale de Kibana (`/etc/kibana/kibana.yml`)

Éditez le fichier de configuration principal :

```bash
sudo vi /etc/kibana/kibana.yml
```

Voici le fichier complet configuré pour votre environnement :

```yaml
# For more configuration options see the configuration guide for Kibana in
# https://www.elastic.co/guide/index.html

# =================== System: Kibana Server ===================
server.port: 5601
server.host: "10.10.40.20"
server.publicBaseUrl: "https://kibana.abest.ovh"
server.name: "plkibana"

# =================== System: Logging ===================
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders:
      - default
      - file

# =================== System: Other ===================
pid.file: /run/kibana/kibana.pid

# =================== Configuration générée et Sécurité ===================
elasticsearch.hosts: ["https://10.10.40.21:9200", "https://10.10.40.22:9200", "https://10.10.40.23:9200", "https://10.10.40.24:9200", "https://10.10.40.25:9200"]
elasticsearch.serviceAccountToken: AAEAAWVsYXN0aWMva2liYW5hL2Vucm9sbC1wcm9jZXNzLXRva2VuLTE3OTMwMjY1MzIwNzU6c29rd3B4N2RUdlNVM1lLanV1SUxydw
elasticsearch.ssl.certificateAuthorities: [/var/lib/kibana/ca_1793026534662.crt]

xpack.fleet.outputs: [{id: fleet-default-output, name: default, is_default: true, is_default_monitoring: true, type: elasticsearch, hosts: [https://10.10.40.21:9200,https://10.10.40.22:9200], ca_trusted_fingerprint: 480b3ec345fdf932ac8d4db3ea7e7294b7837291589d7458ef751549621d485b}]

# Clés de chiffrement des objets et sessions
xpack.encryptedSavedObjects.encryptionKey: 939992b0a163844ad300f5d30a062671d3b263bfa6835df260ed25ee78f63a4b
xpack.reporting.encryptionKey: 057badb15b23f129b98dc0fa95eee2540e527fe0fe9af59bd93f9270351395e6
xpack.security.encryptionKey: 66ea921d4ffd414c6268430d7ddb83fd4f4ec8cf943fe2e3f20cf73ab45be414

# SSL / HTTPS avec certificats Let's Encrypt
server.ssl.enabled: true
server.ssl.certificate: "/etc/kibana/certs/fullchain.pem"
server.ssl.key: "/etc/kibana/certs/privkey.pem"
```

---

## 🛡️ 7. Configuration du Pare-feu (Firewalld)

Autorisez le port `5601/tcp` sur le serveur :

```bash
# Ouverture permanente du port 5601
sudo firewall-cmd --permanent --add-port=5601/tcp

# Rechargement des règles
sudo firewall-cmd --reload

# Vérification
sudo firewall-cmd --list-ports
```

---

## 🔄 8. Démarrage et activation du service Kibana

Rechargez systemd, activez le démarrage automatique et lancez le service :

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kibana
```

Vérifiez que le service fonctionne correctement :

```bash
sudo systemctl status kibana
```

*(Optionnel)* Suivez les logs du démarrage en direct :

```bash
sudo journalctl -u kibana -f
```

---

## 🤖 9. Automations du renouvellement des certificats (Hook Certbot)

Afin d'automatiser la mise à jour des certificats renouvelés par Certbot (expiration tous les 90 jours), créez un script de déploiement automatique :

```bash
sudo vi /etc/letsencrypt/renewal-hooks/deploy/kibana-certs.sh
```

Ajoutez le code suivant :

```bash
#!/bin/bash
DOMAIN="kibana.abest.ovh"

if [ "$RENEWED_LINEAGE" = "/etc/letsencrypt/live/$DOMAIN" ]; then
    cp /etc/letsencrypt/live/$DOMAIN/fullchain.pem /etc/kibana/certs/
    cp /etc/letsencrypt/live/$DOMAIN/privkey.pem /etc/kibana/certs/
    chown -R kibana:kibana /etc/kibana/certs
    chmod 600 /etc/kibana/certs/privkey.pem
    chmod 644 /etc/kibana/certs/fullchain.pem
    systemctl restart kibana
fi
```

Rendez le script exécutable :

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/kibana-certs.sh
```

---

## 🌐 Validation

Accédez à votre serveur via HTTPS depuis un navigateur web :
👉 **`https://kibana.abest.ovh:5601`**
