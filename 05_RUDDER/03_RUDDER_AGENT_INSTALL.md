# Procédure d'installation et de re-initialisation des agents Rudder

Ce document détaille les étapes pour installer, réinitialiser et configurer l'agent **Rudder** sur un serveur (RHEL 9 / compatible RPM), ainsi que les commandes de finalisation (section en attente).

---

## 1. Installation et réinitialisation de l'agent

Cette phase permet d'installer le paquet d'agent Rudder, de nettoyer les anciennes configurations/certificats SSL, puis de réinitialiser la liaison avec le serveur de politiques.

```bash
# Installation du paquet RPM de l'agent Rudder 9.1.5
dnf install http://repository.rudder.io/rpm/9.1/RHEL_9/x86_64/rudder-agent-9.1.5-1.EL.9.x86_64.rpm -y
# Installation du paquet RPM de TAR pour OpenSCAP
dnf install -y tar

# Suppression des anciens fichiers de configuration et certificats SSL
rm -rf /opt/rudder/etc/agent.conf 
rm -f /var/rudder/cfengine-community/ssl/agent.cert 
rm -f /var/rudder/cfengine-community/ssl/ca.cert
rm -f /var/rudder/cfengine-community/ssl/keys/* 
rm -f /var/rudder/cfengine-community/ssl/pinned.pub 

# Réinitialisation de l'agent
rudder agent reset 
rudder agent factory-reset 

# Définition du serveur de politiques (Policy Server)
rudder agent policy-server rudder.abest.ovh 

# Mise à jour des politiques
rudder agent update
```

---

## 2. Configuration HTTPS et enregistrement (Section en attente)

Une fois la pré-installation effectuée, appliquez la configuration HTTPS obligatoire et envoyez le premier inventaire pour validation dans l'interface Rudder.

```bash
# Création du dossier de configuration si nécessaire
sudo mkdir -p /opt/rudder/etc

# Activation de l'option HTTPS uniquement
echo "https_only=true" | sudo tee /opt/rudder/etc/agent.conf

# Confirmation du serveur de politiques
rudder agent policy-server rudder.abest.ovh

# Envoi de l'inventaire initial au serveur Rudder
rudder agent inventory
```

---

> **Note :** Après l'envoi de l'inventaire (`rudder agent inventory`), le nœud apparaîtra dans l'interface web Rudder dans la section **Pending Nodes** (Nœuds en attente) pour acceptation.
