# Dossier Technique
## Samba‑AD - Audit & Remédiation Active Directory

---

## 1. Introduction

Ce dossier présente une analyse approfondie de l’Active Directory basé sur Samba‑AD, incluant :
- Un audit complet PingCastle avant remédiation,
- Une analyse technique détaillée des failles,
- Un plan de remédiation exhaustif,
- Un durcissement AD (Tiering, GPO, ACL, Samba),
- Un audit PingCastle après remédiation,
- Une analyse comparative structurée.

Ce document est conçu comme un livrable professionnel destiné à un RSSI, un DSI ou un auditeur externe.

---

## 2. Contexte & Objectifs

L’infrastructure Active Directory repose sur un contrôleur de domaine Samba‑AD. L’objectif est de :
- Évaluer la posture de sécurité initiale,
- Identifier les failles critiques,
- Appliquer un durcissement complet,
- Mesurer l’impact des remédiations.

L’audit s’appuie sur :
- PingCastle (analyse AD),
- Bonnes pratiques Microsoft ESAE / Tiering Model,
- Durcissement Samba‑AD,
- Durcissement GPO,
- Analyse des ACL,
- Journalisation avancée.

---

## 3. Audit PingCastle – Avant Remédiation

### 3.1 Score Global
- **Score de sécurité** : [à insérer]
- **Risque global** : [à insérer]

### 3.2 Synthèse des Risques
- **Risques critiques** : [à insérer]
- **Risques élevés** : [à insérer]
- **Risques modérés** : [à insérer]

### 3.3 Analyse Technique des Failles

#### 3.3.1 Comptes & Groupes
- Comptes administrateurs présents dans des OU non conformes.
- Absence de séparation des comptes admin (T0/T1/T2).
- Comptes de service avec mots de passe statiques.
- Groupes Domain Admin trop permissifs.

#### 3.3.2 Machines & OU
- Machines Tier2 pouvant administrer Tier1.
- Absence de structure OU adaptée au modèle de tiering.
- Héritage GPO non maîtrisé.

#### 3.3.3 Protocoles & Authentification
- NTLMv1 activé.
- SMB signing non obligatoire.
- LDAP signing non forcé.

#### 3.3.4 GPO
- Absence de GPO de durcissement.
- Absence de LAPS.
- Absence de restrictions PowerShell.

#### 3.3.5 ACL & Délégations
- ACL dangereuses (`WriteDACL`, `GenericAll`).
- Délégations Kerberos non maîtrisées.

#### 3.3.6 Journalisation
- Absence de Sysmon.
- Absence d’audit avancé.
- Absence d’intégration SIEM.

### 3.4 Captures PingCastle
*(Insérer ici les captures du rapport PingCastle avant remédiation)*

---

## 4. Plan de Remédiation – Vue d’Ensemble

Le plan de remédiation est structuré en 7 axes :
1. **Tiering Model** (T0/T1/T2)
2. **Réorganisation des OU**
3. **Séparation des comptes administrateurs**
4. **Durcissement Samba‑AD**
5. **Durcissement GPO**
6. **Nettoyage des ACL & délégations**
7. **Journalisation avancée & SIEM**

---

## 5. Remédiation – Détails Techniques

### 5.1 Mise en place du Tiering Model
- **Tier 0** :
  - DC Samba‑AD
  - Comptes Domain Admin
  - GPO de sécurité du domaine
  - Administration via Jump Server T0
- **Tier 1** :
  - Serveurs applicatifs
  - Comptes Admin T1
  - GPO serveurs
- **Tier 2** :
  - Postes utilisateurs
  - Comptes standards
  - GPO postes

### 5.2 Réorganisation des OU
Création des OU :
```text
OU=Tier0
OU=Tier1
OU=Tier2
OU=Admins
OU=Users
OU=Computers
OU=Groups
```

### 5.3 Séparation des comptes administrateurs
- `admin_t0` → Domain Admin
- `admin_t1` → Admin serveurs
- `admin_t2` → Admin postes

### 5.4 Durcissement Samba‑AD
Modifications du fichier `/etc/samba/smb.conf` :
```ini
ntlm auth = disabled
server signing = mandatory
client ipc signing = mandatory
ldap server require strong auth = yes
```

### 5.5 Durcissement GPO
- **GPO Tier0** :
  - Désactivation NTLMv1
  - Kerberos renforcé
  - Audit avancé
- **GPO Tier1** :
  - Hardening serveurs
  - LAPS
- **GPO Tier2** :
  - Hardening postes
  - Applocker / SRP

### 5.6 Sécurisation des ACL
- Suppression des droits `WriteDACL` non nécessaires
- Blocage des héritages dangereux
- Vérification des délégations Kerberos

### 5.7 Journalisation avancée
- Sysmon
- Audit AD (Événements : 4624, 4625, 4720, 4722, 4732, 4738, 4740…)
- Intégration SIEM

---

## 6. Audit PingCastle – Après Remédiation

### 6.1 Score Global
- **Score de sécurité** : [à insérer]
- **Risque global** : [à insérer]

### 6.2 Améliorations constatées
- Réduction des risques critiques.
- Tiering correctement appliqué.
- GPO durcies et héritage maîtrisé.
- NTLMv1 désactivé.
- SMB signing obligatoire.
- ACL nettoyées.
- Comptes admin correctement isolés.

### 6.3 Captures PingCastle
*(Insérer ici les captures du rapport PingCastle après remédiation)*

---

## 7. Analyse Comparative

| Élément | Avant | Après | Amélioration |
| :--- | :--- | :--- | :---: |
| **Score global** | [x] | [x] | ✔ |
| **Risques critiques** | [x] | [x] | ✔ |
| **NTLMv1** | Activé | Désactivé | ✔ |
| **SMB signing** | Non obligatoire | Obligatoire | ✔ |
| **Tiering** | Absent | Implémenté | ✔ |
| **ACL dangereuses** | Présentes | Corrigées | ✔ |
| **GPO** | Non durcies | Durcies | ✔ |

---

## 8. Conclusion

Le durcissement de l’Active Directory Samba‑AD a permis une amélioration significative de la posture de sécurité. Les résultats PingCastle démontrent une réduction notable des risques critiques et une conformité accrue aux bonnes pratiques de sécurité.

Le modèle de tiering, la séparation des comptes administrateurs, le durcissement Samba, les GPO renforcées et la correction des ACL ont contribué à stabiliser et sécuriser l’environnement AD.

---

## 9. Annexes

- Captures PingCastle avant remédiation
- Captures PingCastle après remédiation
- Commandes `samba-tool` utilisées
- Extraits de configuration `smb.conf`
- Liste des GPO appliquées
