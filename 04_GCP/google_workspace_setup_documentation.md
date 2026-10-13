# Documentation Google Workspace — Inscription & Configuration Domaine

Ce guide détaille les étapes d'inscription, de configuration du domaine et des enregistrements DNS pour **Google Workspace** avec un domaine hébergé sur **OVH** (`abest.ovh`).

---

## 📋 Table des matières
1. [Création du compte Google Workspace](#1-création-du-compte-google-workspace)
2. [Informations sur l'entreprise](#2-informations-sur-lentreprise)
3. [Déclaration du domaine](#3-déclaration-du-domaine)
4. [Choix du domaine pour le compte](#4-choix-du-domaine-pour-le-compte)
5. [Création de l'utilisateur administrateur](#5-création-de-lutilisateur-administrateur)
6. [Première connexion](#6-première-connexion)
7. [Validation du domaine](#7-validation-du-domaine)
8. [Ajout des enregistrements DNS (OVH)](#8-ajout-des-enregistrements-dns-ovh)
9. [Validation réussie](#9-validation-réussie)
10. [Étapes suivantes recommandées](#10-étapes-suivantes-recommandées)

---

## 1. Création du compte Google Workspace

Lors de l’inscription, les informations administratives initiales du compte sont les suivantes :
* **Prénom :** Sylvain
* **Nom :** THOREZ
* **Adresse e‑mail actuelle :** `casmir69@gmail.com`

> ℹ️ **Note :** Ce compte devient automatiquement l’administrateur principal de l’organisation.

---

## 2. Informations sur l’entreprise

Les informations contextuelles requises par Google pour configurer l’organisation Workspace :
* **Nom de l’entreprise :** *[Nom de votre entreprise]*
* **Nombre d’employés :** (inclus vous-même)
* **Région :** France

---

## 3. Déclaration du domaine

* **Domaine professionnel principal :** `abest.ovh`

Google utilise ce domaine pour structurer les adresses e‑mail professionnelles (ex. : `info@abest.ovh`).

---

## 4. Choix du domaine pour le compte

* **Domaine utilisé pour le compte :** `ldap.abest.ovh`

> ℹ️ Tant qu’aucune adresse e-mail personnalisée n’est activement configurée sur le domaine principal, les messages par défaut transitent par ce sous-domaine.

---

## 5. Création de l’utilisateur administrateur

Le premier compte administrateur global de l’organisation a été configuré avec les paramètres suivants :
* **Nom d’utilisateur :** `s.thorez@ldap.abest.ovh`
* **Rôle :** Super Administrateur de l’organisation

---

## 6. Première connexion

Connexion initiale effectuée via l’interface d’administration Google :
* **Identifiant :** `s.thorez@ldap.abest.ovh`

---

## 7. Validation du domaine

Avant d’activer les services Workspace (Gmail, Drive, etc.), Google requiert une preuve de propriété du domaine auprès de l'hébergeur :
* **Hébergeur sélectionné :** `ovh.com`

---

## 8. Ajout des enregistrements DNS (OVH)

Google propose deux méthodes de validation (TXT et CNAME). Les deux ont été implémentées.

### 🔹 Enregistrement TXT
```text
Domaine : ldap
Type : TXT
Valeur : google-site-verification=jJQFtPoZlac9khK78X25a7VsOmXFS9CIMWPLemNqhjk
TTL : 0 (valeur la plus faible)
```

### 🔹 Enregistrement CNAME
```text
Domaine : 4gql4tbnlpua.ldap
Type : CNAME
Cible : gv-pop7vbkqdc3tfw.dv.googlehosted.com
TTL : 0
```

### 🔹 Autres enregistrements présents sur OVH
```text
btelb7nxgxiv.abest.ovh.  -> CNAME -> gv-yyzbmmecbt4sef.dv.googlehosted.com.
abest.ovh.              -> TXT   -> "google-site-verification=NAkhMQWE6PWZ6EMCrfcuVSk-hJU05pcqc1puGvOMJPM"
```

---

## 9. Validation réussie

Google confirme la bonne configuration :
> **Votre domaine a été validé !**
> Vous avez validé `ldap.abest.ovh`

Fonctionnalités désormais débloquées :
* Ajout d’utilisateurs supplémentaires
* Activation des services Workspace (Gmail, Google Drive, Google Meet, Google Chat, etc.)
* Configuration des paramètres de sécurité avancés

---

## 10. Étapes suivantes recommandées

### 1. Configuration des MX Google Workspace (Réception des e-mails)
Ajouter les serveurs de messagerie Google dans la zone DNS OVH :
```text
MX 1 : ASPMX.L.GOOGLE.COM.          (priorité 1)
MX 2 : ALT1.ASPMX.L.GOOGLE.COM.     (priorité 5)
MX 3 : ALT2.ASPMX.L.GOOGLE.COM.     (priorité 5)
MX 4 : ALT3.ASPMX.L.GOOGLE.COM.     (priorité 10)
MX 5 : ALT4.ASPMX.L.GOOGLE.COM.     (priorité 10)
```

### 2. Configuration de l'enregistrement SPF
Empêcher l'usurpation d'identité en déclarant les serveurs autorisés :
```text
v=spf1 include:_spf.google.com ~all
```

### 3. Configuration de DKIM
* À générer depuis la console d'administration Google Workspace (`Admin Console > Applications > Google Workspace > Gmail > Authentifier l'e-mail`).

### 4. Configuration de DMARC
Ajouter la règle DMARC pour le suivi et la gestion des e-mails non authentifiés :
```text
_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:postmaster@abest.ovh"
```