# 📖 Procédure de Post-Installation & Onboarding Serveur (RHEL / Rocky Linux 9)

Cette procédure décrit les étapes permettant de préparer, joindre au domaine Active Directory, sécuriser et enrôler un nouveau serveur Linux (RHEL / Rocky Linux 9) dans la superstructure IT.

---

## 📋 Prérequis

* Un serveur RHEL 9 ou Rocky Linux 9 fraîchement installé.
* Un accès `root` ou privilèges `sudo`.
* Une connectivité réseau vers l'Active Directory, le Proxy Teleport, Rudder et Comodo ITSM.

---

## 🛠️ Étapes de Configuration

### 1. Préparation du Système & Réseau

1. Installez le paquet utilitaire `tar` :
   ```bash
   sudo dnf install tar -y
   ```

2. Configurez les fichiers réseau et la résolution de nom selon les besoins de votre infrastructure :
   * Modifiez le fichier d'hôtes locaux : `/etc/hosts`
   * Définissez le nom d'hôte de la machine : `/etc/hostname`
   * Configurez l'interface réseau principale :
     ```bash
     sudo vi /etc/NetworkManager/system-connections/ens18.nmconnection
     ```

3. Configurez la synchronisation NTP via Chrony (nécessaire pour Kerberos/AD) :
   ```bash
   sudo vi /etc/chrony.conf
   ```
   * *Commentez les pools par défaut et ajoutez vos serveurs NTP internes :*
     ```text
     # pool 2.rocky.pool.ntp.org iburst
     server pldc1.ldap.abest.ovh iburst
     server pldc2.ldap.abest.ovh iburst
     ```

4. Redémarrez le serveur pour appliquer la configuration réseau et NTP :
   ```bash
   sudo reboot
   ```

---

### 2. Jonction au Domaine Active Directory & Droits d'Accès

1. Installez les paquets requis pour l'intégration Active Directory :
   ```bash
   sudo dnf install realmd sssd oddjob oddjob-mkhomedir adcli samba-common samba-common-tools krb5-workstation authselect-compat -y
   ```

2. Joignez le serveur au domaine dans l'OU cible :
   ```bash
   sudo realm join ldap.abest.ovh -U Administrator --computer-ou="OU=LX,OU=Servers,OU=Devices,OU=DSI,OU=ABEST,DC=ldap,DC=abest,DC=ovh"
   ```

3. Restreignez l'accès SSH au groupe Active Directory autorisé :
   ```bash
   sudo realm permit -g SSHSRVADM
   ```

4. Accordez les privilèges Sudo au groupe d'administrateurs AD en créant le fichier `/etc/sudoers.d/domain_admins` :
   ```bash
   echo "%SSHSRVADM@ldap.abest.ovh    ALL=(ALL)    ALL" | sudo tee /etc/sudoers.d/domain_admins
   ```

---

### 3. Installation et Configuration du Nœud Teleport

1. Téléchargez et installez l'agent Teleport :
   ```bash
   sudo dnf install https://cdn.teleport.dev/teleport-18.11.2-1.x86_64.rpm -y
   ```

2. Configurez l'agent Teleport auprès du bastion :
   ```bash
   sudo teleport node configure \
     --token 38c72ded9c0653cdd0c3662e04c4afdb \
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

### 4. Enrôlement de l'Agent de Gestion de Configuration (Rudder)

1. Installez l'agent Rudder pour RHEL 9 :
   ```bash
   sudo dnf install http://repository.rudder.io/rpm/9.1/RHEL_9/x86_64/rudder-agent-9.1.5-1.EL.9.x86_64.rpm -y
   ```

2. Préparez la configuration HTTPS :
   ```bash
   sudo mkdir -p /opt/rudder/etc
   echo "https_only=true" | sudo tee /opt/rudder/etc/agent.conf
   ```

3. Réinitialisez l'état de l'agent si nécessaire et pointez-le vers le serveur de politique :
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

### 5. Installation de l'Agent ITSM / Comodo

1. Réautorisez temporairement l'exécution dans `/tmp` (si la partition est montée en `noexec`) :
   ```bash
   sudo mount -o remount,exec /tmp
   ```

2. Installez `wget` si absent :
   ```bash
   sudo dnf install wget -y
   ```

3. Téléchargez et exécutez l'installeur ITSM :
   ```bash
   cd /tmp
   wget --content-disposition "https://casmir69gmailcom.itsm-us1.comodo.com:443/enroll/linux/run/token/9e8e30bda743be41d08d6b4057f..."
   sudo chmod +x itsm_*_installer.run
   sudo ./itsm_*_installer.run
   ```

4. Vérifiez l'état du service ITSM :
   ```bash
   systemctl status itsm
   ```

---

### 6. Validation & Redémarrage Final

Redémarrez le serveur pour valider la prise en compte de l'ensemble des agents et configurations au démarrage :
```bash
sudo reboot
```
