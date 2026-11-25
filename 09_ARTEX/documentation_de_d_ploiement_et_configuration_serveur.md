# Déploiement d'ARTEX et configuration pare-feu (firewalld)

Cette documentation couvre l'installation et la mise en service d'ARTEX (v0.7.0) via `systemd` ainsi que la configuration du pare-feu `firewalld`.

## 1. Déploiement et service systemd d'ARTEX

### Installation et configuration

```bash
cd /opt/
dnf install -y curl unzip tar ca-certificates nmap nmap-ncat
curl -L -o artex.zip https://github.com/BaaRai/ARTEX/releases/download/v0.7.0/artex-0.7.0-linux-amd64.zip
unzip artex.zip
mv artex-0.7.0-linux-amd64/ artex
cd /opt/artex

cat << 'EOF' > /opt/artex/config.json
{
  "database": {
    "host": "10.10.50.14",
    "port": 5432,
    "user": "artex",
    "password": "<MOT_DE_PASSE_BDD>",
    "dbname": "artexdb",
    "sslmode": "disable"
  },
  "skill_dir": "/opt/artex/skills"
}
EOF

chmod +x artex start.sh
```

> **Sécurité :** Remplacez `<MOT_DE_PASSE_BDD>` par le mot de passe réel de la base PostgreSQL avant le démarrage, et restreignez les permissions du fichier (`chmod 600 /opt/artex/config.json`) si nécessaire.

### Service systemd (`artex.service`)

Si un processus ou script a été lancé manuellement en arrière-plan lors des tests, arrêtez-le avant d'activer `systemd` :

```bash
pkill -f "./start.sh"
pkill -f "/opt/artex/artex"
```

Créez l'unité `/etc/systemd/system/artex.service` en choisissant l'une des deux directives `ExecStart` selon l'exposition réseau souhaitée :

```ini
[Unit]
Description=ARTEX Application Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/artex
ExecStart=/bin/bash /opt/artex/start.sh -addr 127.0.0.1:8787
Restart=always
RestartSec=5
StandardOutput=append:/opt/artex/artex.log
StandardError=append:/opt/artex/artex.log

[Install]
WantedBy=multi-user.target
```

* **Écoute sur toutes les interfaces (`0.0.0.0:8787`)** — nécessaire si des clients ou un reverse proxy distant doivent joindre directement le port `8787` de la machine (à combiner avec `firewalld`) :

```ini
[Unit]
Description=ARTEX Application Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/artex
ExecStart=/bin/bash /opt/artex/start.sh -addr 0.0.0.0:8787
Restart=always
RestartSec=5
StandardOutput=append:/opt/artex/artex.log
StandardError=append:/opt/artex/artex.log

[Install]
WantedBy=multi-user.target
```

Activation et vérification du service :

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now artex
sudo systemctl status artex
tail -n 25 /opt/artex/artex.log
```

## 2. Configuration du pare-feu (`firewalld`)

Vérifiez l'état de `firewalld` et la zone active :

```bash
sudo dnf install -y firewalld
sudo systemctl enable --now firewalld
sudo firewall-cmd --get-active-zones
```

### Cas A : ARTEX écoute sur `0.0.0.0:8787` et doit être accessible depuis le réseau

Ouvrez le port `8787/tcp` de manière permanente dans la zone active (exemple avec `public`) :

```bash
sudo firewall-cmd --zone=public --add-port=8787/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```

### Cas B : Restreindre l'accès au port `8787/tcp` à une IP ou un sous-réseau source

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.10.50.0/24" port protocol="tcp" port="8787" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --list-rich-rules
```

### Cas C : ARTEX écoute uniquement sur `127.0.0.1:8787`

Aucune ouverture du port `8787/tcp` n'est requise dans `firewalld` pour l'accès local à ARTEX.