# 🛠️ Guide : Monter son Lab OpenStack depuis Windows

Ce guide vous accompagne pas à pas pour transformer votre PC Windows en un mini-serveur Cloud OpenStack en utilisant **VirtualBox** et **DevStack**.

---

## 📋 Phase 1 : Préparation de l'hôte (Windows)

### 1. Vérification du Matériel
- **RAM** : Minimum 8 Go (12-16 Go recommandés).
- **BIOS** : Assurez-vous que la **Virtualisation (VT-x ou AMD-V)** est activée dans le BIOS de votre ordinateur.

### 2. Téléchargement des outils
- **VirtualBox** : [Télécharger ici](https://www.virtualbox.org/wiki/Downloads) (Prendre "Windows hosts").
- **Ubuntu 22.04 LTS** : [Télécharger l'ISO Desktop](https://ubuntu.com/download/desktop) (Plus simple pour débuter que la version Server).

---

## 💻 Phase 2 : Création de la Machine Virtuelle (VM)

Ouvrez VirtualBox et cliquez sur **"Nouvelle"** :

1.  **Nom** : `OpenStack-Lab`
2.  **Type** : Linux / Ubuntu (64-bit)
3.  **Mémoire vive (RAM)** : 
    - Minimum : `6144 MB` (6 Go)
    - Recommandé : `8192 MB` (8 Go)
4.  **Disque dur** : Créer un disque virtuel de **50 Go** minimum (Dynamiquement alloué).

### ⚙️ Réglages CRUCIAUX (Avant de démarrer) :
Cliquez sur "Configuration" de votre VM :
- **Système > Processeur** : Donnez **2 CPUs** (ou 4 si vous avez un i7/Ryzen 7).
- **Système > Processeur** : Cochez **"Activer VT-x/AMD-V imbriqué"** (si grisé, ce n'est pas grave, mais c'est mieux).
- **Réseau > Carte 1** : Mode d'accès : **"Accès par pont"** (Bridged Adapter). Cela permet à la VM d'avoir sa propre IP sur votre box internet.
- **Stockage** : Cliquez sur l'icône du CD "Vide" et sélectionnez votre fichier ISO Ubuntu téléchargé.

---

## 🐧 Phase 3 : Installation d'Ubuntu

1.  Démarrez la VM.
2.  Suivez l'installation d'Ubuntu.
3.  **Important** : Choisissez votre nom d'utilisateur (ex: `etudiant`) et un mot de passe simple.
4.  Une fois l'installation finie, retirez le CD virtuel et redémarrez la VM.

---

## 🏗️ Phase 4 : Installation d'OpenStack (DevStack)

Une fois dans Ubuntu, ouvrez un **Terminal** (Ctrl+Alt+T) et tapez ces commandes :

### 1. Préparer l'utilisateur "stack"
OpenStack refuse de s'installer avec votre utilisateur normal ou en root.
```bash
# Créer l'utilisateur stack
sudo useradd -s /bin/bash -d /opt/stack -m stack

# Lui donner les droits d'administration sans mot de passe
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo chmod 0440 /etc/sudoers.d/stack

# Basculer sur l'utilisateur stack
sudo su - stack
```

### 2. Télécharger DevStack
```bash
git clone https://opendev.org/openstack/devstack -b stable/2024.1
cd devstack
```

### 3. Créer le fichier de configuration `local.conf`
Ce fichier dit à OpenStack quel mot de passe utiliser.
```bash
nano local.conf
```
*Copiez-collez ce contenu (clic droit pour coller dans le terminal) :*
```ini
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# Remplacez par l'IP de votre VM Ubuntu (tapez 'hostname -I' pour la connaître)
HOST_IP=192.168.x.x 
```
*(Appuyez sur `Ctrl+O` puis `Entrée` pour enregistrer, et `Ctrl+X` pour quitter)*

### 4. Lancer l'installation
```bash
./stack.sh
```
> ☕ **Pause café** : Cela va prendre entre **20 et 45 minutes** selon votre connexion internet et la puissance de votre PC. Le script va télécharger et configurer des centaines de composants.

---

## 🎉 Phase 5 : Accès au Cloud

À la fin, vous verrez un message avec l'URL d'accès.
1.  Ouvrez le navigateur **sur votre Windows**.
2.  Tapez l'adresse IP de votre VM (ex: `http://192.168.1.15/dashboard`).
3.  **Login** : `admin`
4.  **Password** : `secret` (ou celui que vous avez mis dans local.conf).

---

## 💡 Conseils pour débutant
- **Ne fermez pas la VM brutalement** : Tapez toujours `./unstack.sh` dans le terminal de la VM avant de l'éteindre pour éviter de corrompre la base de données.
- **Snapshot VirtualBox** : Une fois que ça marche, faites un "Instantané" (Snapshot) dans VirtualBox. Si vous cassez quelque chose dans OpenStack, vous pourrez revenir en arrière en 1 clic !
