# 🎓 BOOTCAMP OPENSTACK — JOUR 3 : Administration & Provisioning

---

## 🎯 Objectifs du Jour
- Maîtriser la gestion des identités (Projects, Users, Roles, Quotas)
- Créer une infrastructure complète : Réseau → Routeur → Instances → Volumes
- Comprendre la gestion des images et des flavors
- Manipuler les Floating IPs et Security Groups

---

## ⏱️ Planning du Jour

| Horaire | Session | Type |
|---------|---------|------|
| 09h00 - 10h30 | Module 1 : Gestion des Identités (Keystone) | Théorie |
| 10h30 - 11h00 | ☕ Pause | — |
| 11h00 - 12h30 | Module 2 : Réseau SDN avec Neutron | Théorie |
| 12h30 - 13h30 | 🍽️ Déjeuner | — |
| 13h30 - 15h00 | 🔧 Atelier 3A : Identités et Quotas | Pratique |
| 15h00 - 17h30 | 🔧 Atelier 3B : Infrastructure Complète | Pratique |

---

# MODULE 1 — Gestion des Identités avec Keystone

## 1.1 La Hiérarchie Keystone

```
DOMAIN (organisation)
  └── PROJECT (espace de travail isolé)
        ├── USERS (membres)
        │     └── ROLES (droits)
        └── RESOURCES (VMs, volumes, réseaux...)
```

### Analogie : L'Université

| Concept OpenStack | Analogie Université |
|-------------------|---------------------|
| Domain | L'Université entière |
| Project | Un département (Info, Physique, Maths...) |
| User | Un étudiant ou un professeur |
| Role | Étudiant, Enseignant, Directeur |
| Token | Votre badge d'accès journalier |
| Quota | Le nombre de places en amphi |

## 1.2 Les Rôles Standards

OpenStack 2023+ utilise les **policy enforcements** avec 3 rôles principaux :

| Rôle | Périmètre | Droits |
|------|-----------|--------|
| `admin` | Global | Tout faire sur tout |
| `member` | Project | Gérer les ressources de son projet |
| `reader` | Project | Lire uniquement (audit) |
| `heat_stack_owner` | Project | Créer des stacks Heat |

## 1.3 Les Quotas

Les quotas limitent les ressources par projet. Sans quota, un projet peut consommer toute l'infrastructure.

```bash
# Voir les quotas d'un projet
openstack quota show mon-projet

# Résultat typique :
# +────────────────────────+──────+
# | Ressource              | Max  |
# +────────────────────────+──────+
# | instances              | 10   |
# | cores (vCPU)           | 20   |
# | ram (MB)               | 51200|
# | volumes                | 10   |
# | gigabytes              | 1000 |
# | floating-ips           | 10   |
# | security-groups        | 10   |
# +────────────────────────+──────+

# Modifier les quotas d'un projet
openstack quota set --instances 50 --cores 100 --ram 204800 mon-projet
```

---

# MODULE 2 — Réseau SDN avec Neutron

## 2.1 Concept : Le Réseau Virtuel

> **SDN (Software-Defined Networking)** = Le réseau défini par logiciel.
>
> **Analogie** : Au lieu de brancher des câbles physiques, vous dessinez votre topologie réseau comme sur un tableau blanc, et Neutron la réalise via des logiciels.

## 2.2 Types de Réseaux

### Réseau External (Provider Network)
- Accessible depuis l'extérieur du cloud
- Créé par l'admin uniquement
- Fournit les Floating IPs

```bash
# Création par un admin
openstack network create \
  --external \
  --provider-network-type flat \
  --provider-physical-network physnet1 \
  external-net

openstack subnet create \
  --network external-net \
  --subnet-range 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  --dns-nameserver 8.8.8.8 \
  --no-dhcp \
  external-subnet
```

### Réseau Interne (Tenant Network)
- Privé à un projet
- Isolé des autres projets
- Les VMs y obtiennent des IPs privées via DHCP

```bash
# Création par un utilisateur (dans son projet)
openstack network create reseau-prive

openstack subnet create \
  --network reseau-prive \
  --subnet-range 10.10.10.0/24 \
  --dns-nameserver 8.8.8.8 \
  subnet-prive
```

### Le Routeur Virtuel
- Fait le lien entre le réseau interne et externe
- Permet aux VMs d'accéder à Internet (SNAT)
- Permet l'attribution de Floating IPs (DNAT)

```bash
# Créer un routeur
openstack router create mon-routeur

# Connecter au réseau externe (gateway)
openstack router set --external-gateway external-net mon-routeur

# Connecter au réseau interne (interface)
openstack router add subnet mon-routeur subnet-prive

# Vérifier la topologie
openstack network show mon-routeur
```

## 2.3 Les Floating IPs

```
                    INTERNET
                        │
                   [Routeur OS]
                  /           \
        SNAT (sortant)    DNAT (entrant)
              │                 │
         [VM privée]   ←──  [Floating IP]
         10.10.10.5           192.168.100.50
```

```bash
# Allouer une Floating IP depuis le pool externe
openstack floating ip create external-net

# Associer à une VM
openstack server add floating ip ma-vm 192.168.100.50

# Vérifier
openstack server list --column Name --column Networks
```

## 2.4 Security Groups — Le Firewall par VM

Par défaut : **tout le trafic entrant est bloqué**, tout le trafic sortant est autorisé.

```bash
# Créer un security group dédié web
openstack security group create sg-web --description "Règles pour serveur web"

# SSH depuis partout
openstack security group rule create sg-web \
  --protocol tcp --dst-port 22 --remote-ip 0.0.0.0/0

# HTTP
openstack security group rule create sg-web \
  --protocol tcp --dst-port 80 --remote-ip 0.0.0.0/0

# HTTPS
openstack security group rule create sg-web \
  --protocol tcp --dst-port 443 --remote-ip 0.0.0.0/0

# Ping (ICMP)
openstack security group rule create sg-web \
  --protocol icmp

# Appliquer à une VM
openstack server add security group ma-vm sg-web
```

---

# 🔧 ATELIER 3A — Gestion des Identités et Quotas

## Mission : Créer un Environnement Multi-Tenant

Vous êtes l'admin d'OpenStack pour une entreprise avec 2 équipes.

### Scénario
```
Entreprise : TechCorp
├── Équipe Dev  (project: techcorp-dev)
│     ├── Alice (developpeur, role: member)
│     └── Bob   (tech lead, role: member)
└── Équipe Prod (project: techcorp-prod)
      ├── Carol  (ops, role: member)
      └── Admin  (admin, role: admin)
```

### Étape 1 : Créer les Projects

```bash
# S'assurer d'être en admin
source /etc/kolla/admin-openrc.sh  # ou source /opt/stack/openrc admin admin

# Créer les projets
openstack project create \
  --description "Environnement de développement TechCorp" \
  --domain Default \
  techcorp-dev

openstack project create \
  --description "Environnement de production TechCorp" \
  --domain Default \
  techcorp-prod

# Vérifier
openstack project list
```

### Étape 2 : Créer les Utilisateurs

```bash
# Alice
openstack user create alice \
  --password "Alice2024!" \
  --email alice@techcorp.com

# Bob
openstack user create bob \
  --password "Bob2024!" \
  --email bob@techcorp.com

# Carol
openstack user create carol \
  --password "Carol2024!" \
  --email carol@techcorp.com

# Vérifier
openstack user list
```

### Étape 3 : Assigner les Rôles

```bash
# Alice et Bob dans techcorp-dev
openstack role add --project techcorp-dev --user alice member
openstack role add --project techcorp-dev --user bob member

# Carol dans techcorp-prod
openstack role add --project techcorp-prod --user carol member

# Vérifier
openstack role assignment list --project techcorp-dev --names
```

### Étape 4 : Définir les Quotas

```bash
# Quotas restrictifs pour le dev
openstack quota set techcorp-dev \
  --instances 5 \
  --cores 10 \
  --ram 20480 \
  --volumes 5 \
  --gigabytes 100 \
  --floating-ips 3

# Quotas plus généreux pour la prod
openstack quota set techcorp-prod \
  --instances 20 \
  --cores 40 \
  --ram 81920 \
  --volumes 20 \
  --gigabytes 500 \
  --floating-ips 10

# Vérifier
openstack quota show techcorp-dev
openstack quota show techcorp-prod
```

### Étape 5 : Tester l'Isolation

```bash
# Créer un fichier RC pour Alice
cat > alice-openrc.sh << 'EOF'
export OS_AUTH_URL=http://VOTRE-IP:5000/v3
export OS_PROJECT_NAME=techcorp-dev
export OS_USERNAME=alice
export OS_PASSWORD=Alice2024!
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

# Se connecter en tant qu'Alice
source alice-openrc.sh

# Alice voit-elle les ressources d'autres projets ?
openstack server list  # Doit être vide ou ne montrer que son projet
openstack project list  # Ne doit voir que techcorp-dev
```

---

# 🔧 ATELIER 3B — Infrastructure Complète de A à Z

## Mission : Déployer un Serveur Web Accessible depuis l'Extérieur

Vous allez construire cette topologie :

```
INTERNET
    │
    │ SSH + HTTP
    ▼
[Floating IP 192.168.100.100]
    │
[Routeur virtuel]
    │
[Réseau interne 10.10.10.0/24]
    │
[VM web-server-01 : 10.10.10.5]
    │
[Volume /data : 20 GB]
```

### Étape 1 : Préparer l'Image

```bash
# Télécharger Cirros (image légère pour les tests)
wget http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img

# Uploader dans Glance
openstack image create "Cirros-0.6.2" \
  --file cirros-0.6.2-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public

# Pour Ubuntu (optionnel, image plus lourde)
# wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
# openstack image create "Ubuntu-22.04" \
#   --file jammy-server-cloudimg-amd64.img \
#   --disk-format qcow2 --container-format bare --public

openstack image list
```

### Étape 2 : Créer un Flavor Personnalisé

```bash
# Flavor pour notre web server
openstack flavor create \
  --id auto \
  --vcpus 2 \
  --ram 2048 \
  --disk 20 \
  web.medium

openstack flavor list
```

### Étape 3 : Créer une Paire de Clés SSH

```bash
# Générer une paire de clés
ssh-keygen -t rsa -b 4096 -f ~/.ssh/openstack-bootcamp -N ""

# Importer la clé publique dans OpenStack
openstack keypair create \
  --public-key ~/.ssh/openstack-bootcamp.pub \
  bootcamp-key

# Vérifier
openstack keypair list
```

### Étape 4 : Créer l'Infrastructure Réseau

```bash
# 4.1 Réseau interne
openstack network create reseau-web

openstack subnet create \
  --network reseau-web \
  --subnet-range 10.10.10.0/24 \
  --dns-nameserver 8.8.8.8 \
  --gateway 10.10.10.1 \
  subnet-web

# 4.2 Routeur
openstack router create routeur-web

# Connecter au réseau externe (gateway vers Internet)
openstack router set \
  --external-gateway $(openstack network show external-net -f value -c id) \
  routeur-web

# Connecter au réseau interne
openstack router add subnet routeur-web subnet-web

# Vérifier la topologie réseau
openstack network list
openstack router show routeur-web
```

### Étape 5 : Configurer les Security Groups

```bash
# Créer un security group pour le web server
openstack security group create sg-webserver \
  --description "Web server: SSH + HTTP + ICMP"

# SSH
openstack security group rule create sg-webserver \
  --protocol tcp --dst-port 22 --remote-ip 0.0.0.0/0

# HTTP
openstack security group rule create sg-webserver \
  --protocol tcp --dst-port 80 --remote-ip 0.0.0.0/0

# ICMP (ping)
openstack security group rule create sg-webserver \
  --protocol icmp

openstack security group rule list sg-webserver
```

### Étape 6 : Lancer l'Instance

```bash
openstack server create \
  --image "Cirros-0.6.2" \
  --flavor web.medium \
  --network reseau-web \
  --security-group sg-webserver \
  --key-name bootcamp-key \
  web-server-01

# Surveiller la création
watch -n 2 openstack server show web-server-01 -c status

# Attendre le statut ACTIVE (~1-2 min)
```

### Étape 7 : Associer une Floating IP

```bash
# Allouer une IP depuis le pool externe
FIP=$(openstack floating ip create external-net -f value -c floating_ip_address)
echo "Floating IP allouée : $FIP"

# Associer à notre VM
openstack server add floating ip web-server-01 $FIP

# Vérifier
openstack server list --column Name --column Networks --column Status
```

### Étape 8 : Créer et Attacher un Volume

```bash
# Créer un volume de 20 GB
openstack volume create \
  --size 20 \
  --description "Stockage données web" \
  volume-web-data

# Attendre que le volume soit disponible
openstack volume show volume-web-data -c status

# Attacher à l'instance
openstack server add volume web-server-01 volume-web-data

# Vérifier l'attachement
openstack volume show volume-web-data -c attachments
```

### Étape 9 : Accéder à la VM et Monter le Volume

```bash
# Connexion SSH (Cirros : user=cirros, pass=gocubsgo)
ssh -i ~/.ssh/openstack-bootcamp cirros@$FIP
# Ou pour Ubuntu : ssh -i ~/.ssh/openstack-bootcamp ubuntu@$FIP

# Une fois connecté, vérifier le disque attaché
lsblk
# → Vous devriez voir vdb (le volume Cinder)

# Formater et monter le volume
sudo mkfs.ext4 /dev/vdb
sudo mkdir /data
sudo mount /dev/vdb /data
df -h /data

# Rendre le montage permanent
echo "/dev/vdb /data ext4 defaults 0 2" | sudo tee -a /etc/fstab

# Tester
echo "Hello OpenStack Bootcamp!" | sudo tee /data/test.txt
cat /data/test.txt
```

### Étape 10 : Snapshot — Sauvegarder votre VM

```bash
# Depuis votre station (pas dans la VM)
# 1. Snapshot de l'instance (image de sauvegarde)
openstack server image create \
  --name "web-server-snapshot-$(date +%Y%m%d)" \
  web-server-01

# 2. Snapshot du volume
openstack volume snapshot create \
  --volume volume-web-data \
  --name "vol-snapshot-$(date +%Y%m%d)" \
  --force

# Vérifier
openstack image list --tag snapshot
openstack volume snapshot list
```

---

## 📊 Récapitulatif de l'Infrastructure Créée

```bash
# Vue d'ensemble
echo "=== PROJETS ===" && openstack project list
echo "=== INSTANCES ===" && openstack server list
echo "=== RÉSEAUX ===" && openstack network list
echo "=== VOLUMES ===" && openstack volume list
echo "=== FLOATING IPs ===" && openstack floating ip list
echo "=== IMAGES ===" && openstack image list
```

---

## 📚 Résumé du Jour 3

| Compétence | Acquise ? |
|------------|-----------|
| Créer projects/users/roles | ✅ |
| Gérer les quotas | ✅ |
| Créer des réseaux et sous-réseaux | ✅ |
| Configurer un routeur virtuel | ✅ |
| Utiliser les Security Groups | ✅ |
| Lancer une instance complète | ✅ |
| Attacher et monter un volume | ✅ |
| Créer des snapshots | ✅ |

## 🎯 Pour demain (Jour 4)
- Relisez la topologie que vous avez créée aujourd'hui
- Pensez à un cas d'usage métier que vous aimeriez automatiser
- Préparez vos questions sur les sujets avancés
