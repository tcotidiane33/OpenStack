# 🎓 BOOTCAMP OPENSTACK — JOUR 1 : Architecture & Fondations

---

## 🎯 Objectifs du Jour
À la fin de cette journée, vous serez capable de :
- Expliquer ce qu'est OpenStack et pourquoi il existe
- Identifier et décrire le rôle de chaque composant principal
- Comprendre le flux de communication entre les services
- Naviguer dans l'interface Horizon et utiliser la CLI de base

---

## ⏱️ Planning du Jour

| Horaire | Session | Type |
|---------|---------|------|
| 09h00 - 10h30 | Module 1 : Contexte Cloud et IaaS | Théorie |
| 10h30 - 11h00 | ☕ Pause | — |
| 11h00 - 12h30 | Module 2 : Architecture OpenStack | Théorie |
| 12h30 - 13h30 | 🍽️ Déjeuner | — |
| 13h30 - 15h00 | Module 3 : Composants en détail | Théorie |
| 15h00 - 17h30 | 🔧 Atelier 1 : Premier Contact | Pratique |

---

# MODULE 1 — Contexte Cloud et IaaS

## 1.1 Pourquoi le Cloud ?

### Analogie : Le Courant Électrique
Avant l'électricité publique, chaque usine avait **son propre générateur**. C'était cher, difficile à maintenir, et chaque panne était catastrophique.

Aujourd'hui, vous branchez une prise et vous avez de l'électricité. Vous payez ce que vous consommez.

> **Le Cloud, c'est la même révolution pour l'informatique.**

Au lieu d'acheter des serveurs physiques (générateurs), vous "branchez" vos applications sur une infrastructure mutualisée.

## 1.2 Les Modèles de Services Cloud

```
┌─────────────────────────────────────────────────┐
│  SaaS (Software as a Service)                   │
│  → Gmail, Office 365, Salesforce                │
│  Vous utilisez l'application, c'est tout.       │
├─────────────────────────────────────────────────┤
│  PaaS (Platform as a Service)                   │
│  → Heroku, Google App Engine                    │
│  Vous déployez votre code, la plateforme gère.  │
├─────────────────────────────────────────────────┤
│  IaaS (Infrastructure as a Service)  ← NOUS    │
│  → OpenStack, AWS EC2, Azure VMs                │
│  Vous contrôlez les VMs, réseau, stockage.      │
└─────────────────────────────────────────────────┘
```

## 1.3 OpenStack, c'est quoi exactement ?

OpenStack est une **plateforme open-source de Cloud IaaS** permettant de :
- Créer et gérer des **machines virtuelles** (comme AWS EC2)
- Gérer le **réseau virtuel** (comme AWS VPC)
- Fournir du **stockage** bloc et objet (comme AWS EBS/S3)

**Créé en 2010** par NASA + Rackspace. Aujourd'hui soutenu par des centaines d'entreprises (Red Hat, Canonical, Huawei, OVHcloud...).

---

# MODULE 2 — Architecture OpenStack

## 2.1 L'Analogie de l'Hôtel de Luxe

> Imaginez qu'OpenStack est le **système de gestion d'un grand hôtel**.
> Chaque client veut une chambre (une VM) avec des services spécifiques.

```
🏨 L'HÔTEL OPENSTACK
═══════════════════════════════════════════════════════

 ┌─────────────────┐    ┌──────────────────────────┐
 │   🔐 KEYSTONE   │    │      🌐 HORIZON           │
 │   La Réception  │    │   L'Application Mobile    │
 │   + Sécurité    │    │   de l'Hôtel              │
 └────────┬────────┘    └──────────────────────────┘
          │ (valide tous les badges)
          ▼
 ┌─────────────────────────────────────────────────┐
 │                                                  │
 │  ┌─────────────┐  ┌────────────┐  ┌──────────┐  │
 │  │  🏗️ NOVA   │  │ 📸 GLANCE │  │🔌NEUTRON │  │
 │  │  Le Maître │  │ Catalogue  │  │Plomberie │  │
 │  │  d'Hôtel   │  │ de Déco    │  │& Réseau  │  │
 │  └─────────────┘  └────────────┘  └──────────┘  │
 │                                                  │
 │  ┌─────────────┐  ┌────────────┐                 │
 │  │ 💾 CINDER  │  │ 📦 SWIFT  │                 │
 │  │ Les Coffres│  │ Entrepôt   │                 │
 │  │  Forts     │  │ d'Objets   │                 │
 │  └─────────────┘  └────────────┘                 │
 └─────────────────────────────────────────────────┘
```

## 2.2 Rôle de chaque Composant

### 🔐 Keystone — Le Service d'Identité

**Analogie** : La réception de l'hôtel + le service de sécurité.

- Tout client doit s'authentifier avant d'accéder aux services.
- Il délivre un **token** (badge temporaire) valide pour toutes les requêtes.
- Gère les **Projects** (entreprises clientes), **Users** et **Roles** (permissions).

```
CONCEPTS CLÉS :
├── Domain     → L'organisation globale (ex: votre entreprise)
├── Project    → Un département/équipe (ex: "dev", "prod", "finance")
├── User       → Une personne ou un service
├── Role       → Les droits (admin, member, reader)
└── Token      → Clé d'accès temporaire (valide ~1h par défaut)
```

**Ports** : 5000 (API publique), 35357 (API admin)

---

### 🏗️ Nova — Le Service de Calcul

**Analogie** : Le maître d'hôtel qui attribue et gère les chambres (VMs).

- Reçoit les demandes de création de VMs
- Choisit **quel serveur physique** (nœud compute) hébergera la VM via le **Scheduler**
- Communique avec l'hyperviseur (KVM par défaut) pour démarrer la VM

```
Sous-composants de Nova :
├── nova-api         → Reçoit les requêtes REST
├── nova-scheduler   → Choisit le meilleur nœud compute
├── nova-conductor   → Intermédiaire entre API et compute
├── nova-compute     → Tourne sur chaque hyperviseur, lance les VMs
└── nova-novncproxy  → Accès console VNC aux VMs
```

**Flavors (Gabarits)** : Définissent les ressources d'une VM (CPU, RAM, Disque).

| Flavor | vCPU | RAM | Disque |
|--------|------|-----|--------|
| m1.tiny | 1 | 512 MB | 1 GB |
| m1.small | 1 | 2 GB | 20 GB |
| m1.medium | 2 | 4 GB | 40 GB |
| m1.large | 4 | 8 GB | 80 GB |

---

### 📸 Glance — Le Service d'Images

**Analogie** : Le catalogue de décoration de chambres (modèles prêts à l'emploi).

- Stocke et distribue les **images disques** des VMs (Ubuntu, CentOS, Windows...)
- Nova lui demande l'image au moment de créer une VM
- Supporte plusieurs formats : `qcow2`, `raw`, `vmdk`, `vhd`

```bash
# Exemple : ajouter une image Cirros (image de test légère, ~12MB)
openstack image create "cirros-0.6.2" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

---

### 🔌 Neutron — Le Service Réseau

**Analogie** : La plomberie, l'électricité et le câblage réseau de l'hôtel.

- Fournit du **réseau en tant que service** (Network as a Service - NaaS)
- Crée des réseaux virtuels, sous-réseaux, routeurs, floating IPs
- Utilise des **plugins** (ML2) et **agents** (OVS, LinuxBridge)

```
Composants réseau créés par l'utilisateur :
├── Network    → Le réseau virtuel (ex: "réseau-interne")
├── Subnet     → La plage d'adresses (ex: 192.168.10.0/24)
├── Router     → Pour connecter les réseaux entre eux / vers Internet
├── Port       → La "prise réseau" virtuelle de la VM
└── Floating IP → Adresse IP publique routable depuis l'extérieur
```

---

### 💾 Cinder — Le Service de Stockage Bloc

**Analogie** : Les coffres-forts de l'hôtel — attachables à n'importe quelle chambre.

- Crée des **volumes persistants** (comme des disques durs virtuels)
- Un volume survit à la suppression d'une VM
- Supporte les **snapshots** (photos de l'état d'un volume à un instant T)

```bash
# Créer un volume de 50 Go
openstack volume create --size 50 mon-volume-data

# Attacher à une instance
openstack server add volume ma-vm mon-volume-data
```

---

### 📦 Swift — Le Service de Stockage Objet

**Analogie** : Un entrepôt de fichiers accessible via URL (comme Amazon S3).

- Stockage de fichiers non structurés : images, backups, logs, vidéos
- Accès via API HTTP simple (PUT/GET/DELETE)
- Très scalable, redondant par nature
- Utilisé en interne par Glance pour stocker les images

---

### 🌐 Horizon — Le Dashboard Web

**Analogie** : L'application mobile de l'hôtel — tout piloter depuis son téléphone.

- Interface web complète pour gérer toutes les ressources
- Simplement un **client Python** des APIs OpenStack
- Disponible sur le port **80 (HTTP) ou 443 (HTTPS)**
- Limité par rapport à la CLI pour les opérations avancées

---

## 2.3 Le Flux de Communication — Création d'une VM

Voici ce qui se passe **dans les coulisses** quand vous cliquez "Lancer une instance" :

```
UTILISATEUR
    │
    │ 1. Clique "Créer VM" dans Horizon
    ▼
HORIZON → KEYSTONE : "Ce user/pass sont-ils valides ?"
              │
              └─ KEYSTONE → HORIZON : "Oui, voici votre TOKEN"
    │
    │ 2. Horizon envoie la requête avec le TOKEN
    ▼
NOVA-API (reçoit POST /v2.1/servers)
    │
    │ 3. Valide le token auprès de Keystone
    │ 4. Nova-Scheduler choisit le meilleur nœud compute
    │
    ├──→ GLANCE : "Donne-moi l'image Ubuntu 22.04"
    │        └─ GLANCE → NOVA : "Voici l'URL de l'image"
    │
    ├──→ NEUTRON : "Crée un port réseau pour cette VM"
    │        └─ NEUTRON → NOVA : "Port créé, IP = 10.0.0.5, MAC = fa:16:3e:..."
    │
    ├──→ CINDER (optionnel) : "Attache le volume X"
    │
    └──→ NOVA-COMPUTE (sur le nœud choisi) : "Lance la VM !"
              │
              └─ HYPERVISEUR KVM → VM créée et démarrée ✅
```

---

# MODULE 3 — Les Concepts Essentiels

## 3.1 Le Modèle de Sécurité

### Authentification vs Autorisation
- **Authentification (AuthN)** : "Qui êtes-vous ?" → Géré par Keystone (username/password)
- **Autorisation (AuthZ)** : "Qu'avez-vous le droit de faire ?" → Géré par RBAC (Role-Based Access Control)

### Security Groups
Équivalents à des **firewalls** par VM. Par défaut, tout est bloqué (deny all).

```bash
# Autoriser SSH (port 22) depuis n'importe où
openstack security group rule create --proto tcp --dst-port 22 default

# Autoriser le ping (ICMP)
openstack security group rule create --proto icmp default

# Autoriser HTTP
openstack security group rule create --proto tcp --dst-port 80 default
```

## 3.2 Le Cycle de Vie d'une Instance

```
         PENDING → BUILD → ACTIVE
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
           REBOOT          SHUTOFF         ERROR
              │               │
              └──→ ACTIVE  ←──┘ (power on)
              
              Autres états : PAUSED, SUSPENDED, SHELVED, DELETED
```

| État | Description |
|------|-------------|
| BUILD | La VM est en cours de création |
| ACTIVE | La VM tourne normalement |
| SHUTOFF | La VM est éteinte (RAM libérée, disque conservé) |
| PAUSED | La VM est suspendue en mémoire |
| ERROR | Échec de déploiement |

## 3.3 Endpoints et API

Chaque service expose une **API REST**. Keystone centralise la liste de tous les endpoints :

```bash
# Voir tous les services enregistrés
openstack service list

# Voir tous les endpoints
openstack endpoint list
```

Exemple de réponse :
```
+------------------+----------+----------------------------+
| Name             | Type     | URL                        |
+------------------+----------+----------------------------+
| keystone         | identity | http://controller:5000/v3  |
| nova             | compute  | http://controller:8774/v2.1|
| neutron          | network  | http://controller:9696/v2  |
| glance           | image    | http://controller:9292     |
| cinder           | volume   | http://controller:8776/v3  |
+------------------+----------+----------------------------+
```

---

# 🔧 ATELIER 1 — Premier Contact avec OpenStack

## Prérequis
- Un accès à un environnement OpenStack (DevStack, Horizon ou CLI)
- Le fichier `admin-openrc.sh` pour les credentials

## Étape 1 : Charger les Credentials

```bash
# Option A : Fichier RC (recommandé)
source admin-openrc.sh

# Option B : Variables manuelles
export OS_AUTH_URL=http://controller:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=votremotdepasse
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
```

## Étape 2 : Explorer les Services

```bash
# Vérifier que l'authentification fonctionne
openstack token issue

# Lister les services
openstack service list

# Lister les régions
openstack region list
```

## Étape 3 : Explorer Nova

```bash
# Lister les flavors disponibles
openstack flavor list

# Lister les serveurs existants
openstack server list

# Voir les hyperviseurs disponibles
openstack hypervisor list
```

## Étape 4 : Explorer Glance

```bash
# Lister les images disponibles
openstack image list

# Détails d'une image
openstack image show <nom-ou-id>
```

## Étape 5 : Explorer Neutron

```bash
# Lister les réseaux
openstack network list

# Lister les sous-réseaux
openstack subnet list

# Lister les routeurs
openstack router list
```

## Étape 6 : Explorer la Gestion des Identités

```bash
# Lister les projets
openstack project list

# Lister les utilisateurs
openstack user list

# Lister les rôles
openstack role list
```

## Exercice Final — Mini-Mission 🏆

Créez un nouveau projet et un nouvel utilisateur :

```bash
# 1. Créer un projet
openstack project create --description "Mon projet de test" bootcamp-test

# 2. Créer un utilisateur
openstack user create --password "BootCamp2024!" --email "etudiant@test.com" etudiant-1

# 3. Assigner le rôle "member" à l'utilisateur dans le projet
openstack role add --project bootcamp-test --user etudiant-1 member

# 4. Vérifier
openstack role assignment list --project bootcamp-test --names
```

---

## 📚 Résumé du Jour 1

| Concept | Ce que vous devez retenir |
|---------|--------------------------|
| OpenStack | Plateforme IaaS open-source |
| Keystone | Authentification + Token |
| Nova | Gestion des VMs (Calcul) |
| Glance | Catalogue d'images disque |
| Neutron | Réseau virtuel SDN |
| Cinder | Stockage bloc persistant |
| Swift | Stockage objet (S3-like) |
| Horizon | Dashboard Web |

## 🎯 Pour demain (Jour 2)
- Lire la documentation Neutron sur les types de réseaux (flat, vlan, vxlan)
- Préparer votre VM ou machine physique pour l'installation du Lab
