# 📋 INDEX DU BOOTCAMP OPENSTACK

> Formation complète : Théorie + Ateliers + Montage de Lab

---

## 🗂️ Structure des Fichiers

| Fichier | Contenu | Durée |
|---------|---------|-------|
| [BOOTCAMP_JOUR1_ARCHITECTURE.md](./BOOTCAMP_JOUR1_ARCHITECTURE.md) | Fondations, Architecture, Composants, Analogies | 1 journée |
| [BOOTCAMP_JOUR2_LAB_SETUP.md](./BOOTCAMP_JOUR2_LAB_SETUP.md) | Montage Lab DevStack & Kolla-Ansible AIO | 1 journée |
| [BOOTCAMP_JOUR3_ADMINISTRATION.md](./BOOTCAMP_JOUR3_ADMINISTRATION.md) | Administration, Identités, Réseau SDN, Instances | 1 journée |
| [BOOTCAMP_JOUR4_AVANCE_DEPANNAGE.md](./BOOTCAMP_JOUR4_AVANCE_DEPANNAGE.md) | Heat, Dépannage, Bonne Pratiques Production | 1 journée |

---

## 🧭 Parcours d'Apprentissage Recommandé

```
SEMAINE 1
  Lundi    → Jour 1 : Architecture (Théorie)
  Mardi    → Jour 2 : Montage du Lab
  Mercredi → Révision + Expérimentation libre
  Jeudi    → Jour 3 : Administration
  Vendredi → Jour 4 : Avancé + Projet Final
```

---

## 🚀 Démarrage Rapide (30 min)

### Option A : DevStack sur Ubuntu 22.04

```bash
# 1. Créer l'utilisateur stack
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo su - stack

# 2. Cloner DevStack
git clone https://opendev.org/openstack/devstack -b stable/2024.1
cd devstack

# 3. Configurer (remplacer YOUR_IP)
cat > local.conf << EOF
[[local|localrc]]
ADMIN_PASSWORD=OpenStack2024!
DATABASE_PASSWORD=OpenStack2024!
RABBIT_PASSWORD=OpenStack2024!
SERVICE_PASSWORD=OpenStack2024!
HOST_IP=YOUR_IP
EOF

# 4. Installer (20-40 min)
./stack.sh
```

### Option B : Kolla-Ansible sur Ubuntu 22.04

```bash
# 1. Installer les dépendances
sudo apt update && sudo apt install -y python3-pip python3-venv
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate
pip install 'ansible>=6,<8' kolla-ansible

# 2. Configurer
sudo mkdir -p /etc/kolla && sudo chown $USER /etc/kolla
cp -r $(python3 -c "import kolla_ansible; print(kolla_ansible.__path__[0])")/etc_examples/kolla/* /etc/kolla/
kolla-genpwd

# 3. Éditer /etc/kolla/globals.yml avec votre IP et interface réseau
# Voir BOOTCAMP_JOUR2_LAB_SETUP.md pour la config complète

# 4. Déployer
kolla-ansible -i /etc/kolla/all-in-one bootstrap-servers
kolla-ansible -i /etc/kolla/all-in-one prechecks
kolla-ansible -i /etc/kolla/all-in-one deploy
kolla-ansible -i /etc/kolla/all-in-one post-deploy
source /etc/kolla/admin-openrc.sh
```

---

## 📌 Commandes de Référence Rapide

### Authentification
```bash
source admin-openrc.sh
openstack token issue
```

### Instances (Nova)
```bash
openstack server list
openstack server create --image IMG --flavor FLAVOR --network NET vm-name
openstack server show VM
openstack server delete VM
openstack console log show VM
openstack server reboot --hard VM
```

### Images (Glance)
```bash
openstack image list
openstack image create --file FILE --disk-format qcow2 --public NAME
openstack image show IMAGE
```

### Réseau (Neutron)
```bash
openstack network list
openstack network create NET-NAME
openstack subnet create --network NET --subnet-range 10.0.0.0/24 SUBNET-NAME
openstack router create ROUTER-NAME
openstack router set --external-gateway EXT-NET ROUTER
openstack router add subnet ROUTER SUBNET
openstack floating ip create EXT-NET
openstack server add floating ip VM FIP
```

### Stockage (Cinder)
```bash
openstack volume list
openstack volume create --size 20 VOL-NAME
openstack server add volume VM VOL
openstack volume snapshot create --volume VOL SNAP-NAME
```

### Identités (Keystone)
```bash
openstack project create PROJECT
openstack user create --password PASS USER
openstack role add --project PROJECT --user USER member
openstack quota set --instances 10 PROJECT
```

### Orchestration (Heat)
```bash
openstack stack create -t template.yaml STACK-NAME
openstack stack list
openstack stack show STACK-NAME
openstack stack resource list STACK-NAME
openstack stack output show --all STACK-NAME
openstack stack update -t template.yaml STACK-NAME
openstack stack delete --yes STACK-NAME
```

### Diagnostic
```bash
openstack compute service list
openstack network agent list
openstack volume service list
docker logs CONTAINER --tail 100
docker ps | grep -v pause
```

---

## 🔑 Credentials par Défaut

| Élément | DevStack | Kolla-Ansible |
|---------|----------|---------------|
| Mot de passe admin | ADMIN_PASSWORD dans local.conf | `grep keystone_admin_password /etc/kolla/passwords.yml` |
| Fichier RC | `/opt/stack/openrc admin admin` | `/etc/kolla/admin-openrc.sh` |
| URL Horizon | `http://VOTRE-IP/dashboard` | `http://VOTRE-IP` |
| Login Horizon | `admin` | `admin` |

---

*Bootcamp créé le 27 Avril 2026 — Basé sur OpenStack 2024.1 (Caracal)*
