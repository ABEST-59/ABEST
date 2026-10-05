# Samba‑AD Haute Disponibilité – Documentation Technique
**Plateforme :** Rocky Linux 10  
**Architecture :** DC1 (Master) / DC2 (Secondary) / DC3 (RODC)  
**Domaine :** ldap.abest.ovh | **NetBIOS :** ABEST  

---

## 1. Introduction

Ce dossier technique présente une architecture complète de haute disponibilité (HA) et de résilience pour un domaine **Samba Active Directory (Samba-AD)** sous **Rocky Linux 10**.

Cette documentation traite des sujets suivants :
- Provisionnement du premier contrôleur de domaine (`DC1`).
- Jointure et intégration d'un contrôleur secondaire (`DC2`).
- Déploiement d'un contrôleur de domaine en lecture seule (`DC3 - RODC`).
- Réplication Active Directory (DRS / DSDB / LDB).
- Synchronisation bi-directionnelle / mono-directionnelle de `SYSVOL`.
- Gestion des rôles FSMO (Flexible Single Master Operations).
- Procédures de sauvegardes régulières (Offline, SYSVOL, Secrets).
- Plan de reprise d'activité (Restauration Offline, Bare-Metal, Granulaire).
- Intégration des postes d'administration et outils RSAT.
- Durcissement de la sécurité (`smb.conf`, Kerberos, NTLM) et supervision.

Ce document constitue un livrable d'exploitation à destination des équipes DSI, RSSI et Administrateurs Systèmes & Réseaux.

---

## 2. Installation du DC1 – Premier Contrôleur de Domaine

### 2.1 Pré‑requis et préparation système

#### 2.1.1 Configuration Réseau & Système

| Élément | Valeur |
| :--- | :--- |
| **Système d'exploitation** | Rocky Linux 10 |
| **VLAN** | ADMIN |
| **Adresse IP / Masque** | `10.10.30.10/24` |
| **Passerelle** | `10.10.30.254` |
| **Nom FQDN (Hostname)** | `pldc1.abest.ovh` |
| **Domaine AD (Realm)** | `LDAP.ABEST.OVH` |
| **Nom NetBIOS** | `ABEST` |
| **DNS primaire** | `127.0.0.1` |

#### 2.1.2 Installation avec les infos suivantes

```bash
https://samba.tranquil.it/doc/fr/samba_config_server-server_install_samba_redhat.html
```

### 2.5 Validation DNS et Kerberos

#### 2.5.1 Tests de résolution DNS interne

```bash
# Test des enregistrements SRV LDAP
dig @127.0.0.1 _ldap._tcp.ldap.abest.ovh SRV

# Test des enregistrements SRV Kerberos
dig @127.0.0.1 _kerberos._tcp.ldap.abest.ovh SRV

# Test de résolution directe du domaine
dig @127.0.0.1 ldap.abest.ovh
```

#### 2.5.2 Validation de l'authentification Kerberos

```bash
# Demande de ticket Kerberos pour l'administrateur
kinit administrator@LDAP.ABEST.OVH

# Affichage des tickets enregistrés
klist
```

#### 2.5.3 Verification de l'annuaire LDAP

```bash
ldapsearch -H ldap://127.0.0.1 -x -b "dc=ldap,dc=abest,dc=ovh"
```

#### 2.5.4 Vérification via samba-tool

```bash
samba-tool user list
samba-tool group list
samba-tool drs showrepl
```

---

### 2.6 Configuration recommandée `smb.conf` (DC1)

Éditez le fichier `/etc/samba/smb.conf` :

```ini
[global]
        dns forwarder = 8.8.4.4
#       dns strict mode = yes
        dns zone scavenging = yes

        netbios name = PLDC1
        realm = LDAP.ABEST.OVH
        server role = active directory domain controller
        workgroup = ABEST
        ad dc functional level = 2016

        # disable null session
        restrict anonymous = 2

        # disable netbios
        disable netbios = yes
        smb ports = 445

        # disable printing services
        printcap name = /dev/null
        load printers = no
        disable spoolss = yes
        printing = bsd


        # enable extra hashes
        password hash userPassword schemes = CryptSHA256 CryptSHA512

        # install valid certificate
        tls enabled = yes
        tls keyfile = /etc/samba/tls/key.pem
        tls certfile = /etc/samba/tls/cert.pem
        tls cafile = /etc/samba/tls/ca.pem
        #tls priority = NONE:+SECURE256:-VERS-ALL:+VERS-TLS1.2:+VERS-TLS1.3
        #tls crlfile = /etc/samba/tls/mydomain_authentication.crl
        #tls dhparams file = /etc/samba/tls/srvads.mydomain.lan.dhparams

        ldap server require strong auth = yes

        # enable audit log
        log level = 1 \
          auth_json_audit:3@/var/log/samba/auth_json_audit.log \
          dsdb_json_audit:5@/var/log/samba/dsdb_json_audit.log \
          dsdb_password_json_audit:9@/var/log/samba/dsdb_password_json_audit.log \
          dsdb_group_json_audit:9@/var/log/samba/dsdb_group_json_audit.log \
          kerberos:3@/var/log/samba/kerberos.log \
          dns:0

        # sysvol write log
        full_audit:failure = none
        full_audit:success = pwrite write renameat
        full_audit:prefix = IP=%I|USER=%u|MACHINE=%m|VOLUME=%S
        full_audit:facility = local7
        full_audit:priority = NOTICE


[sysvol]
        path = /var/lib/samba/sysvol
        read only = No
        vfs objects = dfs_samba4, acl_xattr, full_audit

[netlogon]
        path = /var/lib/samba/sysvol/ldap.abest.ovh/scripts
        read only = No
        vfs objects = dfs_samba4, acl_xattr, full_audit
```

---

### 2.7 Intégration d'un Poste d'Administration (PMADM) et outils RSAT

Pour gérer le domaine Samba-AD à distance depuis une machine Windows d'administration (ex: `PMADM`) :

1. Rejoindre la machine Windows au domaine `ldap.abest.ovh`.
2. Installer l'ensemble des fonctionnalités RSAT (Remote Server Administration Tools) via PowerShell en tant qu'Administrateur :

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'RSAT*' | Where-Object State -eq 'NotPresent' | ForEach-Object { Add-WindowsCapability -Online -Name $_.Name }
```

---

### 2.8 Sauvegarde initiale du DC1

#### 2.8.1 Sauvegarde Offline de la base AD (DSDB)

```bash
mkdir -p /root/backup-dc1/secrets
samba-tool domain backup offline --targetdir=/root/backup-dc1/
```

#### 2.8.2 Sauvegarde du volume SYSVOL

```bash
rsync -XAavz /var/lib/samba/sysvol/ /root/backup-dc1/sysvol/
```

#### 2.8.3 Sauvegarde des clés et fichiers secrets

```bash
cp -a /var/lib/samba/private/* /root/backup-dc1/secrets/
```

---

## 3. Installation du DC2 – Haute Disponibilité (Contrôleur Secondaire)

### 3.1 Pré‑requis du DC2

| Élément | Valeur |
| :--- | :--- |
| **OS** | Rocky Linux 10 |
| **Hostname** | `pldc2.abest.ovh` |
| **IP / Masque** | `10.10.30.12/24` |
| **DNS Primaire** | `10.10.30.10` (Pointe vers DC1) |
| **DNS Secondaire** | `127.0.0.1` |

### 3.2 Installation et jointure du DC2 au domaine

```bash
# Installation des paquets
dnf install -y samba samba-dc samba-winbind-clients krb5-workstation bind-utils chrony

# Arrêt et désactivation des services Samba autonomes
systemctl disable --now smb nmb winbind

# Jointure en tant que Contrôleur de Domaine additionnel
samba-tool domain join ldap.abest.ovh DC \
  --realm=LDAP.ABEST.OVH \
  --dns-backend=SAMBA_INTERNAL \
  -U "ABEST\Administrator"
```

### 3.3 Activation du service et synchronisation initiale

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc
```

---

## 4. Installation du DC3 – Read-Only Domain Controller (RODC)

Un RODC est préconisé sur des sites distants ou des zones DMZ/moins sécurisées afin de restreindre le stockage des mots de passe en local.

### 4.1 Pré-requis du DC3

| Élément | Valeur |
| :--- | :--- |
| **OS** | Rocky Linux 10 |
| **Hostname** | `pldc3.abest.ovh` |
| **IP / Masque** | `10.10.30.13/24` |
| **DNS Primaire** | `10.10.30.10` (DC1) |

### 4.2 Jointure en tant que RODC

```bash
samba-tool domain join ldap.abest.ovh RODC \
  --realm=LDAP.ABEST.OVH \
  --dns-backend=SAMBA_INTERNAL \
  -U "ABEST\Administrator"

# Démarrage du service
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc
```

---

## 5. Réplication Active Directory (DRS)

### 5.1 Vérification de l'état de réplication

Exécuter sur n'importe quel contrôleur de domaine :

```bash
samba-tool drs showrepl
```

### 5.2 Forcer la réplication manuelle

Pour forcer une réplication immédiate depuis `DC1` vers `DC2` :

```bash
samba-tool drs replicate pldc2 pldc1 "dc=ldap,dc=abest,dc=ovh"
```

---

## 6. Synchronisation SYSVOL

Samba-AD ne supporte pas nativement FRS/DFSR pour SYSVOL. La synchronisation doit être assurée par un outil externe (Rsync via SSH ou Inotify/Unison).

### 6.1 Synchronisation manuelle via Rsync

Depuis `DC1` vers `DC2` (en préservant les ACLs et attributs étendus XATTR) :

```bash
rsync -XAavz --delete /var/lib/samba/sysvol/ root@10.10.30.12:/var/lib/samba/sysvol/
```

### 6.2 Automatisation par Service et Timer Systemd (DC1)

Créer le fichier `/etc/systemd/system/sysvol-sync.service` :

```ini
[Unit]
Description=Synchronisation SYSVOL vers DC2
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/rsync -XAavz --delete /var/lib/samba/sysvol/ root@10.10.30.12:/var/lib/samba/sysvol/
```

Créer le timer `/etc/systemd/system/sysvol-sync.timer` :

```ini
[Unit]
Description=Timer de synchronisation SYSVOL (Toutes les 5 min)

[Timer]
OnCalendar=*:0/5
Persistent=true

[Install]
WantedBy=timers.target
```

Activer le timer :
```bash
systemctl daemon-reload
systemctl enable --now sysvol-sync.timer
```

---

## 7. Sauvegardes Samba‑AD

### 7.1 Script de Sauvegarde globale (Offline / System)

Un script quotidien sur le serveur `DC1` doit réaliser la sauvegarde complète :

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/samba/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# 1. Sauvegarde Offline AD Database
samba-tool domain backup offline --targetdir="$BACKUP_DIR/ad-db"

# 2. Sauvegarde SYSVOL
rsync -XAavz /var/lib/samba/sysvol/ "$BACKUP_DIR/sysvol/"

# 3. Sauvegarde Secrets & Krb5
cp -p /var/lib/samba/private/randseed.tdb "$BACKUP_DIR/"
cp -p /var/lib/samba/private/dns.keytab "$BACKUP_DIR/"
cp -p /etc/krb5.conf "$BACKUP_DIR/"

echo "Sauvegarde Samba-AD réalisée avec succès dans $BACKUP_DIR"
```

---

## 8. Procédures de Restauration Samba‑AD

### 8.1 Restauration Offline (Bare-Metal)

En cas de perte totale du domaine ou corruption majeure :

1. Stopper les services Samba sur tous les DC :
   ```bash
   systemctl stop samba-ad-dc
   ```
2. Restaurer la base de données Active Directory :
   ```bash
   samba-tool domain backup restore \
     --backupfile=/var/backups/samba/20261005/ad-db/backup-...tar.bz2 \
     --targetdir=/var/lib/samba/restore_db
   ```
3. Remplacer les répertoires d'exploitation :
   ```bash
   rm -rf /var/lib/samba/private /var/lib/samba/sysvol
   cp -a /var/lib/samba/restore_db/private /var/lib/samba/
   cp -a /var/lib/samba/restore_db/sysvol /var/lib/samba/
   ```
4. Vérifier/Réparer les permissions SYSVOL :
   ```bash
   samba-tool ntacl sysvolreset
   ```
5. Redémarrer le service :
   ```bash
   systemctl start samba-ad-dc
   ```

---

## 9. Distribution des Rôles FSMO

Les 5 rôles FSMO (Flexible Single Master Operations) assurent l'unicité de certaines opérations au sein de la forêt et du domaine AD.

### 9.1 Matrice de placement des rôles FSMO

| Rôle FSMO | Portée | Description / Recommandation | Placement Conseillé |
| :--- | :--- | :--- | :--- |
| **Schema Master** | Forêt | Modification du schéma AD | **DC1** |
| **Domain Naming Master** | Forêt | Ajout/Suppression de domaines | **DC1** |
| **PDC Emulator** | Domaine | Gestion du temps, GPO, Mots de passe (Critique) | **DC1** |
| **RID Master** | Domaine | Attribution des RID pour la création d'objets | **DC1** |
| **Infrastructure Master**| Domaine | Références inter-domaines | **DC2** (ou DC1) |

### 9.2 Transfert et Seize (Saisie d'urgence) des rôles FSMO

#### Afficher le propriétaire des rôles :
```bash
samba-tool fsmo show
```

#### Transfert gracieux (ex: vers DC2) :
```bash
samba-tool fsmo transfer --role=all -U "ABEST\Administrator"
```

#### Prise de force (Seize) - uniquement si le DC d'origine est définitivement HS :
```bash
samba-tool fsmo seize --role=all
```

---

## 10. Supervision, Monitoring & Audits

### 10.1 Points de contrôle à intégrer au SIEM / Supervision (Nagios/Zabbix/Prometheus)

1. **Vérification du statut du service :**
   `systemctl is-active samba-ad-dc`
2. **Vérification de la réplication DRS :**
   `samba-tool drs showrepl` (Rechercher l'absence d'erreurs `KCC error` ou `WERR_BADFILE`).
3. **Contrôle d'intégrité de la base LDB :**
   `samba-tool dbcheck`
4. **Journaux Samba :**
   Consulter régulièrement `/var/log/samba/log.samba` et `journalctl -u samba-ad-dc -f`.

---

## 11. Bonnes Pratiques de Sécurité & Durcissement

1. **Minimum 2 Contrôleurs de Domaine fonctionnels** sur le réseau local.
2. **Désactivation totale de NTLMv1** et limitation des algorithmes de chiffrement faibles (ex: RC4).
3. **Signature SMB & LDAP obligatoire** pour contrer les attaques de type *Man-In-The-Middle* (MitM) et *Relay*.
4. **Stratégie d'administration en couches (Tiering Model) :**
   - **Tier 0 :** Contrôleurs de domaine et comptes Admin du domaine.
   - **Tier 1 :** Serveurs membres et applications.
   - **Tier 2 :** Postes de travail et utilisateurs finaux.
5. **Sauvegarde quotidienne Offline** déportée sur un support froid/sécurisé.
6. **Mises à jour de sécurité régulières** du système d'exploitation Rocky Linux et des paquets Samba.

---

## 12. Conclusion

Cette documentation technique définit le socle opérationnel requis pour déployer, administrer et maintenir une infrastructure **Samba Active Directory en Haute Disponibilité** sous **Rocky Linux 10**.

L'association de la réplication native Active Directory (DRS), de la synchronisation du SYSVOL, du respect des règles FSMO et d'un plan rigoureux de sauvegardes garantit une continuité de service maximale face aux pannes logicielles et matérielles.
