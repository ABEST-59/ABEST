# Documentation Technique : Déploiement d'Ollama et du modèle Mistral (CPU)

Cette documentation décrit la procédure complète d'installation, de configuration système et d'optimisation pour héberger le modèle **Mistral (`ministral-3:3b`)** via **Ollama** sur un serveur Linux (basé sur RHEL / Rocky Linux / AlmaLinux / Fedora) fonctionnant sur processeur (CPU).

## 1. Prérequis et Préparation du Système

Avant d'installer Ollama, il est nécessaire d'installer les utilitaires de téléchargement et de décompression requis par le script d'installation officiel (`curl` et `zstd`), puis de préparer l'espace de stockage dédié aux modèles.

### Installation des dépendances

```bash
dnf install -y curl zstd
reboot
```

*(Note : Le redémarrage permet d'appliquer d'éventuelles mises à jour du noyau ou du système avant l'installation).*

### Vérification de l'espace disque

Les modèles d'IA nécessitent plusieurs gigaoctets d'espace disque. Vérifiez la partition `/var` avant de continuer :

```bash
df -h
```

## 2. Installation d'Ollama

Exécutez le script d'installation officiel d'Ollama. Celui-ci télécharge le binaire, crée l'utilisateur système `ollama` et configure le service `systemd`.

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

## 3. Configuration du Service et Stockage des Modèles

Par défaut, Ollama écoute uniquement en local (`127.0.0.1`) et stocke les modèles dans `/usr/share/ollama/.ollama/models`. Pour un environnement serveur de production, il est recommandé de déplacer le stockage vers `/var/ollama-models` et d'ajuster les paramètres de performance CPU/RAM.

### Création du répertoire de stockage

```bash
mkdir -p /var/ollama-models
chown -R ollama:ollama /var/ollama-models
```

### Surcharge de la configuration Systemd (`override.conf`)

Créez le répertoire de surcharge du service et définissez les variables d'environnement :

```bash
mkdir -p /etc/systemd/system/ollama.service.d

cat << 'EOF' > /etc/systemd/system/ollama.service.d/override.conf
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MODELS=/var/ollama-models"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_KEEP_ALIVE=10m"
EOF
```

#### Détail des variables configurées :

* **`OLLAMA_HOST=0.0.0.0:11434`** : Permet d'exposer l'API d'Ollama sur toutes les interfaces réseau du serveur (port `11434`).
* **`OLLAMA_MODELS=/var/ollama-models`** : Définit le répertoire personnalisé où seront téléchargés les poids des modèles.
* **`OLLAMA_NUM_PARALLEL=1`** : Limite le traitement à 1 requête simultanée (évite la saturation du CPU et de la RAM).
* **`OLLAMA_MAX_LOADED_MODELS=1`** : Garde un seul modèle chargé en mémoire vive (RAM) à la fois.
* **`OLLAMA_KEEP_ALIVE=10m`** : Maintient le modèle chargé en RAM pendant 10 minutes après la dernière requête pour accélérer les réponses suivantes, avant de libérer la mémoire.

### Application de la configuration

Rechargez le démon `systemd` et redémarrez le service Ollama :

```bash
systemctl daemon-reload
systemctl restart ollama
```

Vérifiez que le service est bien actif et sans erreur :

```bash
systemctl status ollama --no-pager -l
```

## 4. Configuration du Pare-feu (`firewalld`)

Comme `OLLAMA_HOST=0.0.0.0:11434` expose le service sur le réseau, il faut autoriser le port TCP `11434` dans `firewalld` pour permettre aux machines clientes ou applications externes d'accéder à l'API d'Ollama.

### Vérifier l'état de `firewalld` et la zone active

```bash
systemctl status firewalld --no-pager -l
firewall-cmd --get-active-zones
```

### Ouvrir le port `11434/tcp`

* **Option A — Ouverture sur la zone par défaut (ex. `public`) :**

  ```bash
  firewall-cmd --permanent --add-port=11434/tcp
  firewall-cmd --reload
  ```

* **Option B — Recommandation sécurité : restreindre l'accès à une IP ou un sous-réseau précis (ex. `192.168.10.0/24`) :**

  ```bash
  firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" port protocol="tcp" port="11434" accept'
  firewall-cmd --reload
  ```

### Vérifier les règles actives

```bash
firewall-cmd --list-ports
firewall-cmd --list-rich-rules
```

*(Note : L'API d'Ollama ne gère pas d'authentification native ; évitez d'exposer le port `11434` directement sur Internet sans reverse proxy authentifié / HTTPS ou VPN).*

## 5. Téléchargement et Personnalisation du Modèle Mistral

### Téléchargement du modèle de base

Récupération du modèle léger **Ministral 3B** depuis le registre officiel d'Ollama :

```bash
ollama pull ministral-3:3b
```

### Création du modèle personnalisé (`artex-mistral-cpu`)

Pour adapter le modèle à des tâches nécessitant une mémoire conversationnelle importante (analyse de longs documents, historiques étendus) tout en optimisant le calcul sur processeur, créez un `Modelfile` dédié :

```bash
cat << 'EOF' > /tmp/Modelfile
FROM ministral-3:3b
PARAMETER num_ctx 16384
PARAMETER num_thread 8
EOF
```

#### Explication détaillée des directives du `Modelfile` :

* **`FROM ministral-3:3b`** :
  * **Rôle :** Définit le modèle de base sur lequel s'appuie cette configuration personnalisée.
  * **Explication :** Ollama utilise les poids pré-entraînés du modèle léger *Ministral 3B* (3 milliards de paramètres), particulièrement adapté aux environnements CPU grâce à sa faible empreinte mémoire et sa rapidité d'exécution.

* **`PARAMETER num_ctx 16384`** :
  * **Rôle :** Définit la taille de la fenêtre de contexte (*Context Window*) à **16 384 tokens** (soit environ 12 000 mots ou 25 à 30 pages de texte).
  * **Explication :** Par défaut, Ollama limite le contexte à 2048 ou 4096 tokens. Passer à `16384` permet au modèle de traiter de longs prompts, d'analyser des fichiers entiers ou de conserver un long historique de discussion sans « oublier » le début.
  * **Impact matériel :** Une fenêtre de contexte étendue augmente proportionnellement la consommation de RAM (allocation du *KV Cache*) et peut légèrement ralentir le temps de traitement initial (*prompt evaluation*) sur CPU.

* **`PARAMETER num_thread 8`** :
  * **Rôle :** Fixe à **8** le nombre de threads CPU utilisés simultanément lors de la génération de texte.
  * **Explication :** Ce paramètre doit idéalement correspondre au nombre de **cœurs physiques** (et non logiques/Hyper-Threading) disponibles sur le serveur. Allouer trop de threads par rapport aux cœurs physiques entraîne une contention CPU et dégrade les performances.

### Compilation et mise à jour du modèle

Générez le modèle `artex-mistral-cpu` à partir du `Modelfile` :

```bash
ollama create artex-mistral-cpu -f /tmp/Modelfile
```

> **Bonnes pratiques de mise à jour :** Si vous éditez `/tmp/Modelfile` ultérieurement (via `vi /tmp/Modelfile`), il est impératif de relancer la commande `ollama create artex-mistral-cpu -f /tmp/Modelfile` puis de redémarrer le service (`systemctl restart ollama`) pour recharger les nouveaux paramètres en mémoire.

## 6. Validation, Tests et Supervision

### Vérification des modèles et paramètres

Listez les modèles disponibles sur le serveur :

```bash
ollama list
```

Inspectez les paramètres appliqués au modèle personnalisé pour confirmer que `num_ctx 16384` et `num_thread 8` sont bien actifs :

```bash
ollama show artex-mistral-cpu --parameters
```

### Test fonctionnel d'inférence

Lancez une requête simple en ligne de commande pour valider le bon fonctionnement du modèle :

```bash
ollama run artex-mistral-cpu "Réponds uniquement OK"
```

### Supervision des ressources (CPU, RAM, Disque)

Avec un contexte de `16384` tokens, il est important de surveiller l'empreinte mémoire lors des requêtes longues :

* **Vérifier les modèles actuellement chargés en RAM et leur taille :**

  ```bash
  ollama ps
  ```

* **Surveiller la consommation de mémoire vive (RAM) :**

  ```bash
  free -h
  ```

* **Surveiller l'utilisation CPU en temps réel :**

  ```bash
  top -bn1 | head -n 15
  ```

* **Vérifier l'espace disque restant :**

  ```bash
  df -h
  ```

## 7. Résumé des Commandes de Maintenance

| Action | Commande |
| ----- | ----- |
| **Ouvrir le port 11434 (`firewalld`)** | `firewall-cmd --permanent --add-port=11434/tcp && firewall-cmd --reload` |
| **Lister les ports ouverts (`firewalld`)** | `firewall-cmd --list-ports` |
| **Redémarrer le service Ollama** | `systemctl restart ollama` |
| **Voir les logs du service** | `journalctl -u ollama -f` |
| **Lister les modèles installés** | `ollama list` |
| **Voir les modèles actifs en RAM** | `ollama ps` |
| **Vérifier les paramètres d'un modèle** | `ollama show artex-mistral-cpu --parameters` |
| **Mettre à jour le modèle custom** | `ollama create artex-mistral-cpu -f /tmp/Modelfile` |
| **Tester l'API en local (HTTP)** | `curl http://localhost:11434/api/tags` |
