# Samba‑AD Haute Disponibilité – Documentation Technique
Plateforme : Rocky Linux 10  
Architecture : DC1 / DC2 / DC3 (RODC)

---

## 1. Introduction

Ce document décrit une architecture complète de Samba Active Directory en haute disponibilité, incluant :

- Installation du DC1 (premier contrôleur de domaine)
- Installation du DC2 (contrôleur secondaire répliqué)
- Installation du DC3 RODC
- Réplication AD (DSDB + LDB + SYSVOL)
- Synchronisation SYSVOL
- Mécanismes de bascule
- Sauvegardes (offline, secrets, SYSVOL)
- Procédures de restauration
- Bonnes pratiques de sécurité et supervision

---

## 2. Installation du DC1 – Premier Contrôleur de Domaine

### 2.1 Pré‑requis

#### 2.1.1 Configuration système

| Élément | Valeur |
|--------|--------|
| OS | Rocky Linux 10 |
| VLAN | ADMIN |
| IP | 10.10.30.10/24 |
| Hostname | pldc1.abest.ovh |
| Domaine AD | ldap.abest.ovh |
| NetBIOS | ABEST |
| DNS | 127.0.0.1 |

#### 2.1.2 Installation des paquets

```bash
dnf install samba samba-dc samba-dsdb-modules samba-vfs-modules \
  samba-winbind-clients samba-common-tools krb5-workstation \
  bind-utils chrony -y
