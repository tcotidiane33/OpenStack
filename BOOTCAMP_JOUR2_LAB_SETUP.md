# 🎓 BOOTCAMP OPENSTACK — JOUR 2 : Montage du Lab

---

## 🎯 Objectifs du Jour
- Comprendre les différentes méthodes d'installation
- Monter un lab OpenStack fonctionnel (DevStack ou Kolla-Ansible AIO)
- Comprendre l'architecture réseau du lab
- Valider l'installation et accéder à Horizon

---

## ⏱️ Planning du Jour

| Horaire | Session | Type |
|---------|---------|------|
| 09h00 - 10h00 | Comparaison des méthodes de déploiement | Théorie |
| 10h00 - 11h00 | Architecture réseau du Lab | Théorie |
| 11h00 - 12h30 | 🔧 Atelier 2A : Lab DevStack | Pratique |
| 12h30 - 13h30 | 🍽️ Déjeuner | — |
| 13h30 - 17h30 | 🔧 Atelier 2B : Lab Kolla-Ansible AIO | Pratique |

---

# MODULE 1 — Choisir sa Méthode de Déploiement

## 1.1 Tableau Comparatif

| Critère | DevStack | Kolla-Ansible | OpenStack-Ansible |
|---------|----------|---------------|-------------------|
| **Rapidité** | ⭐⭐⭐⭐⭐ (15-30 min) | ⭐⭐⭐ (1-2h) | ⭐⭐ (2-4h) |
| **Production** | ❌ | ✅ | ✅ |
| **Isolation** | ❌ (bare-metal) | ✅ (Docker) | ❌ |
| **HA possible** | ❌ | ✅ | ✅ |
| **Idéal pour** | Apprendre, CI/CD | Lab avancé, prod | Prod Bare-metal |
| **RAM min** | 8 GB | 16 GB | 32 GB |

## 1.2 Recommandation Pédagogique

```
DÉBUTANT     → DevStack (PC/VM avec 8-16 GB RAM)
INTERMÉDIAIRE → Kolla-Ansible AIO (VM avec 16-32 GB RAM)
AVANCÉ       → Kolla-Ansible Multi-nodes (Cluster physique/VMs)
```

---

# MODULE 2 — Architecture Réseau du Lab

## 2.1 Les 3 Types de Réseaux dans OpenStack

```
┌─────────────────────────────────────────────────────┐
│         ARCHITECTURE RÉSEAU D'UN LAB                │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Management Network (eth0)  10.0.0.0/24     │   │
│  │  → Communication entre les services OS      │   │
│  │  → Accès SSH admin                          │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Data/Tunnel Network (eth1) 10.0.1.0/24     │   │
│  │  → Trafic entre VMs (VXLAN tunnels)         │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  External Network (eth2) 10.0.2.0/24        │   │
│  │  → Floating IPs, accès Internet des VMs     │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

> **Pour un lab sur une seule machine** : On peut utiliser une seule interface et simuler les réseaux avec des bridges virtuels (c'est ce que fait DevStack).

## 2.2 Topologie All-in-One (AIO)

```
MACHINE PHYSIQUE ou VM
┌──────────────────────────────────────────────────┐
│                                                  │
│  ┌────────────────────────────────────────────┐ │
│  │          HYPERVISEUR (KVM)                 │ │
│  │  ┌──────────────────────────────────────┐  │ │
│  │  │       VM OPENSTACK AIO              │  │ │
│  │  │  ┌──────────┐  ┌────────────────┐  │  │ │
│  │  │  │ KEYSTONE │  │ NOVA, GLANCE,  │  │  │ │
│  │  │  │ NEUTRON  │  │ CINDER, SWIFT  │  │  │ │
│  │  │  │ HORIZON  │  │    RABBITMQ    │  │  │ │
│  │  │  └──────────┘  └────────────────┘  │  │ │
│  │  └──────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

---

# 🔧 ATELIER 2A — Lab DevStack (Méthode Rapide)

## Prérequis Matériels

```
✅ CPU    : 4+ vCPU avec virtualisation (VT-x/AMD-V)
✅ RAM    : Minimum 8 GB (recommandé 16 GB)
✅ Disque : 100 GB minimum (SSD recommandé)
✅ OS     : Ubuntu 22.04 LTS (fresh install)
✅ Réseau : Accès Internet
```

## Étape 1 : Préparer le Système Ubuntu 22.04

```bash
# 1. Mise à jour complète du système
sudo apt update && sudo apt upgrade -y

# 2. Installer les dépendances de base
sudo apt install -y git curl wget vim python3-pip net-tools

# 3. Désactiver le swap (recommandé pour OpenStack)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 4. Vérifier que la virtualisation imbriquée est disponible
egrep -c '(vmx|svm)' /proc/cpuinfo
# Si > 0 : OK, si = 0 : KVM ne fonctionnera pas (utiliser QEMU)
```

## Étape 2 : Créer un Utilisateur Dédié

DevStack ne doit **jamais** être exécuté en root.

```bash
# Créer l'utilisateur 'stack'
sudo useradd -s /bin/bash -d /opt/stack -m stack

# Lui donner les droits sudo sans mot de passe
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo chmod 0440 /etc/sudoers.d/stack

# Basculer sur l'utilisateur stack
sudo su - stack
```

## Étape 3 : Cloner DevStack

```bash
# Cloner le dépôt DevStack (branche stable recommandée)
git clone https://opendev.org/openstack/devstack -b stable/2024.1
cd devstack

# Vérifier la version
cat .git/HEAD
```

## Étape 4 : Créer la Configuration (local.conf)

```bash
# Obtenir l'IP de votre machine
hostname -I | awk '{print $1}'
# → Notez cette IP (ex: 192.168.56.10)

# Créer le fichier de configuration
cat > local.conf << 'EOF'
[[local|localrc]]

# ─── Mots de passe ───────────────────────────────────
ADMIN_PASSWORD=OpenStack2024!
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# ─── Réseau ──────────────────────────────────────────
# Remplacez par votre IP réelle
HOST_IP=192.168.56.10

# Réseau externe pour les Floating IPs
FLOATING_RANGE=192.168.56.0/24
PUBLIC_NETWORK_GATEWAY=192.168.56.1
PUBLIC_INTERFACE=eth0

# ─── Services à activer ──────────────────────────────
ENABLED_SERVICES=rabbit,mysql,key
ENABLED_SERVICES+=,n-api,n-cond,n-sch,n-cpu,n-novnc,n-api-meta
ENABLED_SERVICES+=,placement-api,placement-client
ENABLED_SERVICES+=,g-api,g-reg
ENABLED_SERVICES+=,q-svc,q-agt,q-dhcp,q-l3,q-meta
ENABLED_SERVICES+=,c-api,c-vol,c-bak
ENABLED_SERVICES+=,horizon

# ─── Image de base ───────────────────────────────────
IMAGE_URL_SITE="http://download.cirros-cloud.net"
IMAGE_URL_PATH="/0.6.2/"
IMAGE_URL_FILE="cirros-0.6.2-x86_64-disk.img"
IMAGE_URLS="$IMAGE_URL_SITE$IMAGE_URL_PATH$IMAGE_URL_FILE"

# ─── Paramètres optionnels ───────────────────────────
LOGFILE=$DEST/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True
SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5
SWIFT_REPLICAS=1
SWIFT_DATA_DIR=$DEST/data/swift

EOF

echo "✅ local.conf créé !"
```

## Étape 5 : Lancer l'Installation

```bash
# L'installation prend entre 20 et 45 minutes
./stack.sh

# En cas d'interruption, pour nettoyer et recommencer :
./unstack.sh && ./clean.sh && ./stack.sh
```

### Que faire pendant l'installation ?
Suivez les logs en temps réel dans un second terminal :
```bash
sudo su - stack
tail -f /opt/stack/logs/stack.sh.log
```

## Étape 6 : Valider l'Installation

```bash
# Vérifier que tous les services sont actifs
source /opt/stack/openrc admin admin
openstack service list
openstack compute service list
openstack network agent list

# Test rapide : créer une image test
openstack image list
```

## Étape 7 : Accéder à Horizon

```
URL     : http://VOTRE-IP/dashboard
Login   : admin
Password: OpenStack2024! (votre ADMIN_PASSWORD)
```

---

# 🔧 ATELIER 2B — Lab Kolla-Ansible AIO (Méthode Docker)

## Prérequis Matériels

```
✅ CPU    : 8+ vCPU avec virtualisation (VT-x/AMD-V)
✅ RAM    : Minimum 16 GB (recommandé 32 GB)
✅ Disque : 100 GB minimum + 50 GB pour Cinder LVM
✅ OS     : Ubuntu 22.04 LTS
✅ Réseau : Accès Internet + 2 interfaces réseau (ou 1 + loopback)
✅ Docker : Sera installé par Kolla
```

## Étape 1 : Préparer le Système

```bash
# Mise à jour
sudo apt update && sudo apt upgrade -y

# Dépendances Python
sudo apt install -y python3-dev libffi-dev gcc libssl-dev python3-pip python3-venv

# Créer un environnement virtuel Python
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate

# Vérifier
python3 --version  # >= 3.8
pip --version
```

## Étape 2 : Installer Kolla-Ansible

```bash
source /opt/kolla-venv/bin/activate

# Installer Ansible + Kolla-Ansible
pip install 'ansible>=6,<8'
pip install kolla-ansible

# Vérifier
kolla-ansible --version
ansible --version
```

## Étape 3 : Configurer Kolla

```bash
# Créer le répertoire de configuration
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# Copier les fichiers de config par défaut
cp -r $(python3 -c "import kolla_ansible; print(kolla_ansible.__path__[0])")/etc_examples/kolla/* /etc/kolla/
cp $(python3 -c "import kolla_ansible; print(kolla_ansible.__path__[0])")/ansible/inventory/all-in-one /etc/kolla/
```

## Étape 4 : Générer les Mots de Passe

```bash
# Génération automatique de tous les mots de passe
kolla-genpwd

# Vérifier
grep "keystone_admin_password" /etc/kolla/passwords.yml
```

## Étape 5 : Configurer globals.yml

```bash
# Récupérer le nom de l'interface réseau principale
ip a | grep "state UP" | awk '{print $2}' | sed 's/://'
# → ex: eth0, ens3, enp3s0

# Récupérer votre IP
hostname -I | awk '{print $1}'

# Éditer la configuration principale
cat > /etc/kolla/globals.yml << 'EOF'
---
# ─── Base ────────────────────────────────────────────
kolla_base_distro: "ubuntu"
openstack_release: "2024.1"

# ─── Réseau ──────────────────────────────────────────
# Remplacez eth0 par votre interface réelle
network_interface: "eth0"
neutron_external_interface: "eth0"  # Même interface pour AIO

# IP de management (votre IP principale)
kolla_internal_vip_address: "VOTRE-IP"

# ─── Neutron ─────────────────────────────────────────
neutron_plugin_agent: "openvswitch"
neutron_type_drivers: "flat,vlan,vxlan"
neutron_tenant_network_types: "vxlan"

# ─── Stockage Cinder (LVM) ───────────────────────────
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"

# ─── Services optionnels ─────────────────────────────
enable_haproxy: "no"   # Non nécessaire pour AIO
enable_heat: "yes"     # Orchestration
enable_horizon: "yes"  # Dashboard web

# ─── OpenStack Tags ──────────────────────────────────
nova_compute_virt_type: "kvm"  # ou "qemu" si pas de VT-x natif
EOF
```

## Étape 6 : Préparer le Groupe de Volumes Cinder

```bash
# Créer un fichier loopback comme "disque" pour Cinder (si pas de 2ème disque)
sudo dd if=/dev/zero of=/var/lib/cinder-volumes.img bs=1M count=50000
sudo losetup /dev/loop0 /var/lib/cinder-volumes.img

# Créer le Physical Volume et Volume Group LVM
sudo pvcreate /dev/loop0
sudo vgcreate cinder-volumes /dev/loop0

# Vérifier
sudo vgs
```

## Étape 7 : Bootstrap et Pré-vérifications

```bash
source /opt/kolla-venv/bin/activate

# Installer les dépendances Ansible sur le nœud cible
kolla-ansible -i /etc/kolla/all-in-one bootstrap-servers

# Vérifier les prérequis (DOIT passer sans erreur)
kolla-ansible -i /etc/kolla/all-in-one prechecks
```

## Étape 8 : Déploiement (≈ 30-60 min)

```bash
# Lancer le déploiement
kolla-ansible -i /etc/kolla/all-in-one deploy

# En cas de succès, vous verrez :
# PLAY RECAP *****
# localhost : ok=XXX changed=YYY unreachable=0 failed=0
```

## Étape 9 : Post-Déploiement

```bash
# Générer le fichier de credentials admin
kolla-ansible -i /etc/kolla/all-in-one post-deploy

# Source des variables d'environnement
source /etc/kolla/admin-openrc.sh

# Vérifier les services
openstack service list
openstack compute service list
openstack network agent list
```

## Étape 10 : Initialiser les Ressources de Base

```bash
# Script fourni par Kolla pour créer les ressources de démo
KOLLA_PATH=$(python3 -c "import kolla_ansible; print(kolla_ansible.__path__[0])")
$KOLLA_PATH/tools/init-runonce

# Ce script crée automatiquement :
# ✅ Un réseau external (public)
# ✅ Un réseau interne (demo-net)
# ✅ Un routeur reliant les deux
# ✅ Une image Cirros de test
# ✅ Une paire de clés SSH
# ✅ Des security group rules de base
```

---

## ✅ Checklist de Validation du Lab

```bash
# 1. Services Nova
openstack compute service list
# → Tous en état "up" ✅

# 2. Agents Neutron
openstack network agent list
# → Tous en état "alive" ✅

# 3. Images disponibles
openstack image list
# → Cirros présent ✅

# 4. Réseaux créés
openstack network list
openstack subnet list
openstack router list
# → public, private, router présents ✅

# 5. Test ultime : Lancer une VM !
openstack server create \
  --image cirros \
  --flavor m1.tiny \
  --network private \
  --key-name mykey \
  test-vm-01

# Attendre et vérifier le statut
watch openstack server list
# → STATUS doit passer à ACTIVE ✅
```

---

## 🔍 Dépannage des Problèmes Courants

| Problème | Cause probable | Solution |
|----------|----------------|----------|
| `prechecks` échoue | Docker non démarré | `sudo systemctl start docker` |
| "No valid host found" | Pas de VG Cinder | Vérifier `sudo vgs` |
| VM en état ERROR | VT-x pas disponible | Passer `nova_compute_virt_type: "qemu"` |
| Horizon inaccessible | HAProxy ou Kolla | `docker ps \| grep horizon` |
| IP non attribuée | Neutron agent down | `openstack network agent list` |

```bash
# Voir les logs d'un service en cas d'erreur
docker logs nova_compute --tail 50
docker logs neutron_server --tail 50
docker logs keystone --tail 50

# Reconfigurer après une modification
kolla-ansible -i /etc/kolla/all-in-one reconfigure
```

---

## 📚 Résumé du Jour 2

| Décision | DevStack | Kolla-Ansible AIO |
|----------|----------|-------------------|
| Apprendre vite | ✅ | — |
| Lab durable | — | ✅ |
| Proche de la prod | — | ✅ |
| RAM disponible 8 GB | ✅ | — |
| RAM disponible 16+ GB | ✅ | ✅ |

## 🎯 Pour demain (Jour 3)
- Votre lab doit être opérationnel
- Accédez à Horizon et explorez l'interface
- Préparez-vous à créer votre première infrastructure complète
