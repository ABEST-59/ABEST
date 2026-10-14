# SAMBA-AD : Forcer l’utilisation de LDAPS

## 1. Objectif

Ce document décrit la procédure complète pour :
- Activer LDAPS sur deux contrôleurs de domaine Samba‑AD (`pldc1` et `pldc2`)
- Installer et configurer Let’s Encrypt via DNS‑01 OVH
- Déployer les certificats TLS dans Samba
- Forcer les postes Windows à utiliser LDAPS au lieu de LDAP
- Désactiver le fallback LDAP non chiffré

## 2. Environnement

- **Distribution** : Rocky Linux 9/10
- **Contrôleurs de domaine Samba‑AD** :
  - `pldc1.ldap.abest.ovh`
  - `pldc2.ldap.abest.ovh`
- **DNS provider** : OVH (DNS‑01)
- **Objectif sécurité** : LDAPS obligatoire pour tous les clients Windows

## 3. Configuration Let’s Encrypt (DNS‑01 OVH)

### 3.1 Installation Certbot + plugin OVH
```bash
dnf install certbot python3-certbot-dns-ovh -y
```

### 3.2 Création du fichier de credentials OVH
Créer : `/etc/letsencrypt/ovh.ini`

Contenu :
```ini
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = <APP_KEY>
dns_ovh_application_secret = <APP_SECRET>
dns_ovh_consumer_key = <CONSUMER_KEY>
```

Sécurisation :
```bash
chmod 600 /etc/letsencrypt/ovh.ini
```

## 4. Génération des certificats Let’s Encrypt

### 4.1 pldc1
```bash
certbot certonly --dns-ovh --dns-ovh-credentials /etc/letsencrypt/ovh.ini -d pldc1.ldap.abest.ovh
```

### 4.2 pldc2
```bash
certbot certonly --dns-ovh --dns-ovh-credentials /etc/letsencrypt/ovh.ini -d pldc2.ldap.abest.ovh
```

Les certificats sont générés dans : `/etc/letsencrypt/live/pldc2.ldap.abest.ovh/`

## 5. Déploiement des certificats dans Samba

### 5.1 pldc1
```bash
mkdir -p /etc/samba/tls
cp /etc/letsencrypt/live/pldc1.ldap.abest.ovh/fullchain.pem /etc/samba/tls/cert.pem
cp /etc/letsencrypt/live/pldc1.ldap.abest.ovh/privkey.pem /etc/samba/tls/key.pem
cp /etc/letsencrypt/live/pldc1.ldap.abest.ovh/chain.pem /etc/samba/tls/ca.pem
chmod 600 /etc/samba/tls/key.pem
ls -la /etc/samba/tls
```

### 5.2 pldc2
```bash
mkdir -p /etc/samba/tls
cp /etc/letsencrypt/live/pldc2.ldap.abest.ovh/fullchain.pem /etc/samba/tls/cert.pem
cp /etc/letsencrypt/live/pldc2.ldap.abest.ovh/privkey.pem /etc/samba/tls/key.pem
cp /etc/letsencrypt/live/pldc2.ldap.abest.ovh/chain.pem /etc/samba/tls/ca.pem
chmod 600 /etc/samba/tls/key.pem
ls -l /etc/samba/tls
```

## 6. Configuration Samba pour LDAPS

Modifier `/etc/samba/smb.conf` :
```ini
[global]
    tls enabled = yes
    tls keyfile = /etc/samba/tls/key.pem
    tls certfile = /etc/samba/tls/cert.pem
    tls cafile = /etc/samba/tls/ca.pem
    ldap server require strong auth = yes
```

Redémarrer Samba :
```bash
systemctl restart samba
```

## 7. Forcer Windows à utiliser LDAPS

### 7.1 GPO : LDAP Signing
Chemin : `GPO` → `Computer Configuration` → `Policies` → `Windows Settings` → `Security Settings` → `Local Policies` → `Security Options`

Activer :
- `Domain controller: LDAP server signing requirements = Require signing`
- `Network security: LDAP client signing requirements = Require signing`

### 7.2 GPO : Désactiver le fallback LDAP
- `Domain controller: Allow LDAP fallback = Disabled`

### 7.3 GPO : Sécurisation Kerberos / NTLM
- `Network security: LAN Manager authentication level = Send NTLMv2 only`
- `Network security: Do not store LAN Manager hash value = Enabled`

## 8. Tests LDAPS

### 8.1 Test LDAP via TLS
```bash
ldapsearch -H ldaps://pldc1.ldap.abest.ovh -x -LLL -b "dc=ldap,dc=abest,dc=ovh"
ldapsearch -H ldaps://pldc2.ldap.abest.ovh -x -LLL -b "dc=ldap,dc=abest,dc=ovh"
```

### 8.2 Test TLS brut
```bash
openssl s_client -connect pldc1.ldap.abest.ovh:636 -showcerts
openssl s_client -connect pldc2.ldap.abest.ovh:636 -showcerts
```

### 8.3 Vérification réplication AD
```bash
samba-tool drs showrepl
```

## 9. Renouvellement automatique

Créer `/usr/local/sbin/samba-renew.sh` :
```bash
#!/bin/bash
certbot renew --dns-ovh --quiet
cp /etc/letsencrypt/live/pldc1.ldap.abest.ovh/fullchain.pem /var/lib/samba/private/tls/cert.pem
cp /etc/letsencrypt/live/pldc1.ldap.abest.ovh/privkey.pem /var/lib/samba/private/tls/key.pem
cp /etc/letsencrypt/live/pldc1.ldap.abest.ovh/chain.pem /var/lib/samba/private/tls/ca.pem
systemctl restart samba-ad-dc
```

Configurer le Cron (à adapter pour `pldc2`) :
```cron
0 3 * * * root /usr/local/sbin/samba-renew.sh
```

## 10. Résultat final

Après application :
- LDAPS activé sur `pldc1` et `pldc2`
- Certificats Let’s Encrypt valides
- LDAP non chiffré désactivé
- Windows forcé à utiliser LDAPS
- Domaine sécurisé et conforme aux bonnes pratiques

## 11. Annexes

### Ports utilisés
- **LDAP** : 389 (désactivé)
- **LDAPS** : 636

### Chemins Samba
- `/var/lib/samba/private/tls/cert.pem`
- `/var/lib/samba/private/tls/key.pem`
- `/var/lib/samba/private/tls/ca.pem`
