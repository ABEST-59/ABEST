# Teleport : Installation & Configuration du Serveur Bastion

Ce document décrit la procédure étape par étape pour installer et configurer un serveur **Teleport (version 18.11.2)** sur un système basé sur RHEL/Rocky Linux, son intégration Active Directory, la configuration du pare-feu et son raccordement au cluster PostgreSQL.

---

## Table des matières

1. [Intégration Active Directory (realmd)](#1-intégration-active-directory-realmd)
2. [Installation de Teleport](#2-installation-de-teleport)
3. [Configuration du Firewall](#3-configuration-du-firewall)
4. [Configuration de Teleport (`/etc/teleport.yaml`)](#4-configuration-de-teleport-etcteleportyaml)
5. [Démarrage et activation du service](#5-démarrage-et-activation-du-service)

---

## 1. Intégration Active Directory (realmd)

### Installation des paquets requis

```bash
sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation authselect-compat -y
```

### Jonction au domaine AD

```bash
sudo realm join ldap.abest.ovh -U Administrator --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=ABEST DSI,DC=ldap,DC=abest,DC=ovh"
```

### Restriction des accès SSH

Autoriser uniquement les membres du groupe AD `SSHSRVADM` à se connecter :

```bash
sudo realm permit -g SSHSRVADM
```

### Configuration des privilèges Sudo

Créez ou éditez le fichier `/etc/sudoers.d/domain_admins` :

```bash
sudo vi /etc/sudoers.d/domain_admins
```

Ajoutez la ligne suivante pour accorder les droits sudo aux administrateurs du domaine :

```text
# AD group (prefix with %)
%SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL
```

---

## 2. Installation de Teleport

Télécharger et installer le paquet RPM officiel de Teleport :

```bash
# Téléchargement du paquet RPM
curl -O https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm

# Installation du paquet via DNF
sudo dnf install https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm -y
```

---

## 3. Configuration du Firewall

Autoriser le trafic HTTPS sur le port `443/TCP` (utilisé par le Proxy Web Teleport et ACME Let's Encrypt) :

```bash
# Vérifier les ports actuellement ouverts
sudo firewall-cmd --list-ports

# Autoriser le port HTTPS (443/TCP) de manière permanente
sudo firewall-cmd --add-port=443/tcp --permanent

# Recharger la configuration du pare-feu
sudo firewall-cmd --reload

# Vérifier la prise en compte du port
sudo firewall-cmd --list-ports
```

---

## 4. Configuration de Teleport (`/etc/teleport.yaml`)

Éditez le fichier de configuration principal `/etc/teleport.yaml` :

```bash
sudo vi /etc/teleport.yaml
```

Remplacez son contenu par la configuration ci-dessous :

```yaml
version: v3
teleport:
  nodename: plbastion
  data_dir: /var/lib/teleport
  join_params:
    token_name: ""
    method: token
  log:
    output: stderr
    severity: INFO
    format:
      output: text
  ca_pin: ""
  diag_addr: ""

  # Stockage de l'état du cluster dans le serveur PostgreSQL distant
  storage:
    type: postgres
    conn_string: "postgres://teleport:123456789AD@postgres-bastion.abest.ovh:5432/teleportdb?sslmode=verify-full"

auth_service:
  enabled: "yes"
  listen_addr: 0.0.0.0:3025
  cluster_name: bastion.abest.ovh
  proxy_listener_mode: multiplex

ssh_service:
  enabled: "yes"
  labels:
    env: production

proxy_service:
  enabled: "yes"
  web_listen_addr: 0.0.0.0:443
  public_addr: bastion.abest.ovh:443
  https_keypairs: []
  https_keypairs_reload_interval: 0s
  acme:
    enabled: "yes"
    email: s.thorez@outlook.fr
```

> **Remarque :** Assurez-vous que le serveur PostgreSQL `postgres-bastion.abest.ovh` est bien accessible depuis ce serveur et que le certificat TLS est valide pour la connexion `sslmode=verify-full`.

---

## 5. Démarrage et activation du service

Activez le service au démarrage du système et démarrez-le immédiatement :

```bash
sudo systemctl enable --now teleport
```

Pour vérifier le statut du service et consulter les logs :

```bash
# Vérifier l'état du service
sudo systemctl status teleport

# Consulter les logs en temps réel
sudo journalctl -fu teleport
```