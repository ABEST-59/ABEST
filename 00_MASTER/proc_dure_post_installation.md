# 📖 Procédure de Post-Installation & Onboarding Serveur (RHEL / Rocky Linux 9)

Cette procédure décrit les étapes permettant de préparer, mettre à jour, joindre au domaine Active Directory, sécuriser et enrôler un nouveau serveur Linux (RHEL / Rocky Linux 9) dans l'infrastructure.

## 📋 Prérequis

* Un serveur RHEL 9 ou Rocky Linux 9 fraîchement installé.
* Un accès `root` ou privilèges `sudo`.
* Une connectivité réseau vers l'Active Directory, le Proxy Teleport, Rudder et WAPT.

---

## 🛠️ Étapes de Configuration

### 1. Mise à Jour du Système & Services SSH

1. Mettez à jour le système et installez le client/serveur SSH ainsi que l'utilitaire `tar` :
   ```bash
   sudo dnf update -y
   sudo dnf install tar -y
   ```

2. Ajustez la configuration SSH si nécessaire :
   ```bash
   sudo vi /etc/ssh/sshd_config
   sudo systemctl restart sshd
   ```

---

### 2. Analyse de Sécurité & Conformité (OpenSCAP)

1. Installez les outils d'analyse de vulnérabilités et de conformité OpenSCAP :
   ```bash
   sudo dnf install -y openscap-scanner scap-security-guide
   ```

---

### 3. Jonction au Domaine Active Directory & Droits d'Accès

1. Installez les paquets requis pour l'intégration Active Directory :
   ```bash
   sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation -y
   ```

2. Joignez le serveur au domaine dans l'OU cible (saisissez le mot de passe administrateur `$$ToTo.com11@@` lorsque demandé) :
   ```bash
   sudo realm join ldap.abest.ovh -U Administrator --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=DSI,OU=ABEST,DC=ldap,DC=abest,DC=ovh"
   ```

3. Restreignez l'accès SSH au groupe Active Directory autorisé et configurez les droits sudo :
   ```bash
   sudo realm permit -g SSHSRVADM
   echo "%SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL" | sudo tee /etc/sudoers.d/domain_admins
   ```

---

### 4. Installation et Configuration du Nœud Teleport

1. Téléchargez et installez l'agent Teleport :
   ```bash
   sudo dnf install https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm -y
   ```

2. Configurez l'agent Teleport auprès du bastion :
   ```bash
   sudo teleport node configure \
     --token 3e67fe5b86c8146dd530e3bf6d369da7 \
     --ca-pin sha256:bf8a6db4179690ed87ba788089818ab4c5ac12c2d3e352a93554b256aed4668d \
     --proxy bastion.abest.ovh:443 \
     --labels Infra=PRD \
     --output /etc/teleport.yaml
   ```

3. Activez et démarrez le service Teleport :
   ```bash
   sudo systemctl enable --now teleport
   ```

---

### 5. Enrôlement de l'Agent de Gestion de Configuration (Rudder)

1. Installez l'agent Rudder pour RHEL 9 :
   ```bash
   sudo dnf install http://repository.rudder.io/rpm/9.1/RHEL_10/x86_64/rudder-agent-9.1.5-1.EL.10.x86_64.rpm -y
   ```

2. Préparez la configuration HTTPS :
   ```bash
   sudo mkdir -p /opt/rudder/etc
   echo "https_only=true" | sudo tee /opt/rudder/etc/agent.conf
   ```

3. Réinitialisez l'état de l'agent et pointez-le vers le serveur de politique :
   ```bash
   sudo rudder agent reset
   sudo rudder agent factory-reset
   sudo rudder agent policy-server rudder.abest.ovh
   ```

4. Envoyez le premier inventaire et appliquez les politiques :
   ```bash
   sudo rudder agent inventory -v
   sudo rudder agent update -v
   ```

---

### 6. Installation et Configuration de l'Agent WAPT

1. Installez l'agent WAPT :
   ```bash
   sudo dnf install https://wapt.abest.ovh/wapt/waptagent/tis-waptagent.x86_64.rpm -y
   ```

2. Configurez l'agent, générez son UUID et redémarrez le service :
   ```bash
   sudo wapt-get reset-config-from-url https://wapt.abest.ovh//wapt/conf.d/default_1f1479ed483a8ca354f0886c44dccd0524156a8bc4141e559eec8744bd474991.json
   sudo wapt-get generate-uuid
   sudo systemctl restart waptservice
   ```

---

### 7. Configuration du Pare-feu (Firewalld)

1. Installez, activez et configurez Firewalld pour autoriser le trafic entrant sur le port 443 :
   ```bash
   sudo dnf install -y firewalld && sudo systemctl enable --now firewalld
   sudo firewall-cmd --permanent --add-port=443/tcp
   sudo firewall-cmd --reload
   ```

---

### 8. Validation & Redémarrage Final

Redémarrez le serveur pour valider la prise en compte de l'ensemble des agents, règles de sécurité et configurations au démarrage :
```bash
sudo reboot
```
