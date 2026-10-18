# Procédure d'installation et de configuration de PLRUDDER (RHEL 9)

Ce document regroupe l'ensemble des étapes de configuration système, d'intégration Active Directory, de déploiement de Teleport, d'installation de Rudder 9.1, de génération de certificat SSL via DNS OVH et de publication applicative dans Teleport.

---

## Table des matières

1. [Configuration Réseau & System](#1-configuration-réseau--system)
2. [Jonction au Domaine Active Directory (AD)](#2-jonction-au-domaine-active-directory-ad)
3. [Droits Sudo pour les Administrateurs AD](#3-droits-sudo-pour-les-administrateurs-ad)
4. [Installation & Configuration de Teleport Node](#4-installation--configuration-de-teleport-node)
5. [Installation de Rudder Server 9.1](#5-installation-de-rudder-server-91)
6. [Configuration du Pare-feu (Firewalld)](#6-configuration-du-pare-feu-firewalld)
7. [Administration Rudder](#7-administration-rudder)
8. [Certificat SSL Let's Encrypt (DNS OVH)](#8-certificat-ssl-lets-encrypt-dns-ovh)
9. [Configuration Apache VirtualHost pour Rudder](#9-configuration-apache-virtualhost-pour-rudder)
10. [Configuration SSL dans Rudder Web](#10-configuration-ssl-dans-rudder-web)
11. [Publication d'Application dans Teleport](#11-publication-dapplication-dans-teleport)

---

## 1. Configuration Réseau & System

1. Éditer le fichier `/etc/hosts` :
   ```bash
   sudo vi /etc/hosts
   ```

2. Configurer le nom d'hôte (`/etc/hostname`) :
   ```bash
   sudo vi /etc/hostname
   ```

3. Définir la configuration réseau de la carte `ens18` :
   ```bash
   sudo vi /etc/NetworkManager/system-connections/ens18.nmconnection
   ```

4. Redémarrer le système pour valider les paramètres réseau :
   ```bash
   sudo reboot
   ```

---

## 2. Jonction au Domaine Active Directory (AD)

1. Installer les paquets nécessaires à la jonction AD :
   ```bash
   sudo dnf install -y realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation authselect-compat
   ```

2. Rejoindre le domaine `ldap.abest.ovh` dans l'OU cible :
   ```bash
   sudo realm join ldap.abest.ovh \
     -U Administrator \
     --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=DSI,OU=ABEST,DC=ldap,DC=abest,DC=ovh"
   ```

3. Autoriser uniquement les membres du groupe AD `SSHSRVADM` à se connecter :
   ```bash
   sudo realm permit -g SSHSRVADM
   ```

---

## 3. Droits Sudo pour les Administrateurs AD

1. Éditer la configuration Sudo :
   ```bash
   sudo vi -f /etc/sudoers.d/domain_admins
   ```

2. Ajouter la ligne suivante :
   ```text
   # AD group (prefix with %)
   %SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL
   ```

3. Ajuster les permissions du fichier :
   ```bash
   sudo chmod 440 /etc/sudoers.d/domain_admins
   ```

---

## 4. Installation & Configuration de Teleport Node

1. Télécharger et installer l'agent Teleport :
   ```bash
   sudo dnf install -y https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm
   ```

2. Générer le fichier de configuration de l'agent Teleport :
   ```bash
   sudo teleport node configure \
     --token 9ea591d56abbbcf7abab54eb68970dd4 \
     --ca-pin sha256:bf8a6db4179690ed87ba788089818ab4c5ac12c2d3e352a93554b256aed4668d \
     --proxy bastion.abest.ovh:443 \
     --labels Infra=PRD \
     --output /etc/teleport.yaml
   ```

3. Activer et démarrer le service Teleport :
   ```bash
   sudo systemctl enable --now teleport
   ```

4. Redémarrer le nœud :
   ```bash
   sudo reboot
   ```

---

## 5. Installation de Rudder Server 9.1

1. Installer le paquet de la base de données externe :
   ```bash
   sudo dnf install -y http://repository.rudder.io/rpm/9.1/RHEL_9/noarch/rudder-external-db-9.1.5-1.EL.9.noarch.rpm
   ```

2. Sauvegarder la configuration par défaut de la BDD :
   ```bash
   sudo mv /opt/rudder/etc/external-db.conf /opt/rudder/etc/external-db.old
   ```

3. Installer l'agent, le relais et le serveur Rudder :
   ```bash
   sudo dnf install -y http://repository.rudder.io/rpm/9.1/RHEL_9/x86_64/rudder-agent-9.1.5-1.EL.9.x86_64.rpm
   sudo dnf install -y http://repository.rudder.io/rpm/9.1/RHEL_9/x86_64/rudder-relay-9.1.5-1.EL.9.x86_64.rpm
   sudo dnf install -y http://repository.rudder.io/rpm/9.1/RHEL_9/x86_64/rudder-server-9.1.5-1.EL.9.x86_64.rpm
   ```

---

## 6. Configuration du Pare-feu (Firewalld)

1. Déplacer l'interface `ens18` dans la zone `internal` :
   ```bash
   sudo firewall-cmd --permanent --zone=internal --change-interface=ens18
   ```

2. Autoriser le port HTTPS (443/TCP) dans la zone `internal` :
   ```bash
   sudo firewall-cmd --permanent --zone=internal --add-port=443/tcp
   ```

3. Recharger la configuration du pare-feu :
   ```bash
   sudo firewall-cmd --reload
   ```

---

## 7. Administration Rudder

1. Créer un utilisateur administrateur pour l'interface web Rudder :
   ```bash
   sudo rudder server create-user -u Administrator
   ```

---

## 8. Certificat SSL Let's Encrypt (DNS OVH)

1. Installer Certbot et le plugin DNS OVH :
   ```bash
   sudo dnf install -y certbot python3-certbot-dns-ovh
   ```

2. Créer le fichier d'API OVH :
   ```bash
   sudo vi /root/ovh.ini
   ```

3. Définir des droits stricts sur le fichier d'API OVH :
   ```bash
   sudo chmod 600 /root/ovh.ini
   ```

4. Générer le certificat Let's Encrypt via challenge DNS :
   ```bash
   sudo certbot certonly --dns-ovh --dns-ovh-credentials /root/ovh.ini -d rudder.abest.ovh
   ```

---

## 9. Configuration Apache VirtualHost pour Rudder

1. Éditer le fichier de configuration Apache de Rudder :
   ```bash
   sudo vi /etc/httpd/conf.d/rudder.conf
   ```

2. Remplacer ou appliquer le bloc de configuration VirtualHost suivant :
   ```apache
   #SINGLE_VHOST_START
   # This default virtual host serves (on root servers) both Web/API and node communication
   # using the agent certificate.
   # See below for other configuration options.
   <VirtualHost *:443>
     ServerAdmin webmaster@localhost
     ServerName rudder.abest.ovh
     ServerAlias rudder.bastion.abest.ovh

     SSLEngine on

     # Bind your Let's Encrypt files here for public Web/API access:
     SSLCertificateFile /etc/letsencrypt/live/rudder.abest.ovh/fullchain.pem
     SSLCertificateKeyFile /etc/letsencrypt/live/rudder.abest.ovh/privkey.pem

     LogLevel warn
     CustomLog /var/log/rudder/apache2/access.log combined
     ErrorLog /var/log/rudder/apache2/error.log

     # Include relay configuration, plus webapp configuration if root server
     # Configuration to use the agent certificate, required for node-server communication,
     # is included in rudder-apache-relay-ssl.conf
     Include         /opt/rudder/etc/rudder-apache-relay-ssl.conf
     IncludeOptional /opt/rudder/etc/rudder-apache-webapp-ssl.conf
   </VirtualHost>
   #SINGLE_VHOST_END
   ```

3. Valider puis redémarrer le service Apache HTTPD :
   ```bash
   sudo apachectl configtest
   sudo systemctl restart httpd
   ```

---

## 10. Configuration SSL dans Rudder Web

1. Éditer le fichier des propriétés Web de Rudder :
   ```bash
   sudo vi /opt/rudder/etc/rudder-web.properties
   ```

2. Modifier ou ajouter les paramètres HTTPS :
   ```properties
   rudder.server.certificate.httpsOnly=true

   # EN ATTENTE DE VALIDATION :
   # rudder.server.hsts=true
   # rudder.server.hstsIncludeSubDomains=true
   ```

---

## 11. Publication d'Application dans Teleport

1. Générer un token d'application court terme depuis la machine bastion (`plbastion`) :
   ```bash
   tctl tokens add --type=app --ttl=5m
   ```

2. Éditer le fichier `/etc/teleport.yaml` sur la machine PLRUDDER :
   ```bash
   sudo vi /etc/teleport.yaml
   ```

3. Mettre à jour la ligne `token` avec le jeton généré et ajouter le bloc `app_service` :
   ```yaml
   app_service:
     enabled: yes
     apps:
       - name: opnsense
         uri: "https://rudder.abest.ovh/rudder"
         public_addr: "rdduer.bastion.abest.ovh"
         insecure_skip_verify: true
         labels:
           env: PRD
   ```

4. Redémarrer Teleport pour valider la configuration :
   ```bash
   sudo systemctl restart teleport
   ```
