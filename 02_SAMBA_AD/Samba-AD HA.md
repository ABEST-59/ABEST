Plateforme : Rocky Linux 10  
Architecture : DC1 / DC2 / DC3 (RODC)

1. Introduction
Ce document décrit une architecture complète de Samba Active Directory en haute disponibilité, incluant :

Installation du DC1 (premier contrôleur de domaine)

Installation du DC2 (contrôleur secondaire répliqué)

Installation du DC3 RODC

Réplication AD (DSDB + LDB + SYSVOL)

Synchronisation SYSVOL

Mécanismes de bascule

Sauvegardes (offline, secrets, SYSVOL)

Procédures de restauration

Bonnes pratiques de sécurité et supervision

Ce livrable est destiné aux équipes DSI / RSSI / Exploitation.

2. Installation du DC1 – Premier Contrôleur de Domaine
2.1 Pré‑requis
2.1.1 Configuration système
Élément	Valeur
OS	Rocky Linux 10
VLAN	ADMIN
IP	10.10.30.10/24
Hostname	pldc1.abest.ovh
Domaine AD	ldap.abest.ovh
NetBIOS	ABEST
DNS	127.0.0.1


2.1.2 Installation des paquets
bash
dnf install samba samba-dc samba-dsdb-modules samba-vfs-modules \
  samba-winbind-clients samba-common-tools krb5-workstation \
  bind-utils chrony -y
2.1.3 Désactivation des services Samba classiques
bash
systemctl disable --now smb nmb winbind
2.2 Préparation du système
2.2.1 Nettoyage éventuel
bash
rm -rf /var/lib/samba/*
rm -rf /etc/samba/smb.conf
2.2.2 Activation NTP
bash
systemctl enable --now chronyd
2.3 Provisionnement du domaine
2.3.1 Création du domaine
bash
samba-tool domain provision \
  --use-rfc2307 \
  --realm=LDAP.ABEST.OVH \
  --domain=ABEST \
  --server-role=dc \
  --dns-backend=SAMBA_INTERNAL
2.3.2 Installation du fichier Kerberos
bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
2.4 Activation du service Samba‑AD
bash
systemctl enable --now samba
systemctl status samba
2.5 Tests DNS
2.5.1 Test SRV LDAP
bash
dig @127.0.0.1 _ldap._tcp.ldap.abest.ovh SRV
2.5.2 Test résolution domaine
bash
dig @127.0.0.1 ldap.abest.ovh
2.6 Tests Kerberos
bash
kinit administrator
klist
2.7 Test LDAP
bash
ldapsearch -H ldap://127.0.0.1 -x -b "dc=ldap,dc=abest,dc=ovh"
2.8 Vérification Samba‑tool
bash
samba-tool user list
samba-tool group list
samba-tool drs showrepl
2.9 smb.conf recommandé (DC1)
ini
[global]
    server role = active directory domain controller
    workgroup = ABEST
    realm = LDAP.ABEST.OVH
    dns forwarder = 8.8.4.4

    # Sécurité
    ntlm auth = disabled
    server signing = mandatory
    client ipc signing = mandatory
    ldap server require strong auth = yes
Niveau fonctionnel AD
bash
samba-tool domain level raise --domain-level=2016 --forest-level=2016
2.10 Sauvegarde initiale du DC1
2.10.1 Sauvegarde offline
bash
samba-tool domain backup offline --targetdir=/root/backup-dc1/
2.10.2 Sauvegarde SYSVOL
bash
rsync -XAavz /var/lib/samba/sysvol/ /root/backup-dc1/sysvol/
2.10.3 Sauvegarde secrets
bash
cp /var/lib/samba/private/* /root/backup-dc1/secrets/
3. Installation du DC2 – Haute Disponibilité
3.1 Pré‑requis
Élément	Valeur
Hostname	pldc2.ldap.abest.ovh
IP	10.10.30.12
DNS	DC1


3.2 Installation des paquets
bash
dnf install samba samba-dc samba-winbind-clients krb5-workstation bind-utils -y
3.3 Joindre le domaine en tant que DC
bash
samba-tool domain join LDAP.ABEST.OVH DC \
  --realm=LDAP.ABEST.OVH \
  --dns-backend=SAMBA_INTERNAL \
  --username=administrator
3.4 Activation du service
bash
systemctl enable --now samba
4. Réplication Active Directory
4.1 Vérification
bash
samba-tool drs showrepl
4.2 Forcer la réplication
bash
samba-tool drs replicate pldc2 pldc1 "dc=ldap,dc=abest,dc=ovh"
5. Synchronisation SYSVOL
5.1 Synchronisation rsync + inotify
bash
rsync -XAavz --delete /var/lib/samba/sysvol/ pldc2:/var/lib/samba/sysvol/
5.2 Automatisation via systemd
Créer un service + timer.

6. Sauvegardes Samba‑AD
6.1 Sauvegarde offline
bash
samba-tool domain backup offline --targetdir=/root/backup/
6.2 Sauvegarde SYSVOL
bash
rsync -XAavz /var/lib/samba/sysvol/ /root/backup/sysvol/
6.3 Sauvegarde secrets
bash
cp /var/lib/samba/private/* /root/backup/secrets/
7. Restauration Samba‑AD
7.1 Restauration offline
bash
samba-tool domain backup restore \
  --backupdir=/root/backup/ \
  --targetdir=/var/lib/samba/restore/
7.2 Restauration SYSVOL
bash
rsync -XAavz /root/backup/sysvol/ /var/lib/samba/sysvol/
7.3 Restauration Kerberos
bash
cp /root/backup/secrets/* /var/lib/samba/private/
8. Rôles FSMO – Placement recommandé
Rôle FSMO	Recommandation	Placement
Schema Master	Rare	DC1
Domain Naming Master	Rare	DC1
PDC Emulator	Critique	DC1
RID Master	Avec PDC	DC1
Infrastructure Master	Pas sur GC	DC2


9. Supervision & Monitoring
Logs Samba : /var/log/samba/

Réplication : samba-tool drs showrepl

Kerberos : kinit, klist

Intégration SIEM : auditd, journald, logs AD

10. Bonnes Pratiques de Sécurité
Minimum 2 DC par site

Sauvegarde quotidienne offline

Synchronisation SYSVOL automatisée

Durcissement smb.conf

Désactivation NTLMv1

Signing obligatoire

Tiering AD (T0/T1/T2)

GPO de durcissement

11. Conclusion
Cette architecture fournit une solution Samba‑AD robuste, sécurisée et hautement disponible, adaptée à un environnement professionnel.
