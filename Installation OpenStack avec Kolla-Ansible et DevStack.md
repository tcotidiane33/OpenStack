# Installation OpenStack avec Kolla-Ansible et DevStack

## Introduction aux concepts

### Kolla-Ansible vs DevStack

**Kolla-Ansible** est un outil de déploiement de production OpenStack qui utilise:
- Des containers Docker pour isoler les services
- Ansible pour l'orchestration
- Une architecture scalable et haute disponibilité

**DevStack** est un environnement de développement qui:
- Installe OpenStack directement sur le système hôte
- Est conçu pour le développement et les tests
- N'est pas adapté pour la production

## Cas d'utilisation typiques

| Scénario               | Kolla-Ansible | DevStack |
|------------------------|---------------|----------|
| Environnement de prod  | ✓             | ✗        |
| Développement OpenStack| ✓             | ✓        |
| Tests CI/CD            | ✓             | ✓        |
| POC rapide             | ✗             | ✓        |
| Apprentissage          | ✓             | ✓        |

## Installation avec Kolla-Ansible

### 1. Prérequis système

```bash
# Sur Ubuntu 22.04
sudo apt update
sudo apt install -y python3-dev libffi-dev gcc libssl-dev python3-pip
sudo pip3 install -U pip
```

### 2. Installation des dépendances

```bash
sudo pip3 install ansible==6.6.0
sudo pip3 install kolla-ansible==15.1.0
```

### 3. Configuration de l'inventaire

```bash
sudo mkdir -p /etc/kolla
sudo cp -r /usr/local/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
sudo cp /usr/local/share/kolla-ansible/ansible/inventory/* /etc/kolla/
```

### 4. Configuration de Kolla

Éditez `/etc/kolla/globals.yml`:

```yaml
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "zed"
kolla_internal_vip_address: "10.0.0.100"
network_interface: "eth0"
neutron_external_interface: "eth1"
enable_haproxy: "yes"
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_cinder_backend_lvm: "yes"
```

### 5. Préparation de l'environnement

```bash
sudo kolla-ansible -i /etc/kolla/all-in-one bootstrap-servers
sudo kolla-ansible -i /etc/kolla/all-in-one prechecks
```

### 6. Déploiement

```bash
sudo kolla-ansible -i /etc/kolla/all-in-one deploy
```

### 7. Post-déploiement

```bash
sudo kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
```

## Installation avec DevStack

### 1. Préparation du système

```bash
sudo apt update
sudo apt install -y git
git clone https://opendev.org/openstack/devstack
cd devstack
```

### 2. Configuration minimale

Créez un fichier `local.conf`:

```ini
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD
```

### 3. Exécution de DevStack

```bash
./stack.sh
```

## Architecture des services et flux

### Architecture Kolla-Ansible

```
[Client]
  |
  v
[HAProxy] (Load Balancer)
  |
  +--> [Nova-API] (Docker Container)
  |     |
  |     v
  |   [Nova-Conductor]
  |     |
  |     v
  |   [Nova-Compute] (Sur nœud compute)
  |
  +--> [Neutron-Server]
        |
        +--> [OVS Agent] (Sur chaque nœud)
        |
        +--> [L3 Agent]  (Pour le routing)
```

### Flux typique - Création d'une instance

1. **Authentification**: Client -> Keystone
2. **Demande de création**: Client -> Nova-API
3. **Orchestration**: Nova-API -> Nova-Conductor -> Nova-Compute
4. **Réseau**: Nova-Compute -> Neutron (allocation IP, ports)
5. **Stockage**: Nova-Compute -> Cinder (volumes) ou Glance (images)
6. **Exécution**: Nova-Compute -> Hyperviseur (KVM généralement)

### Services clés et leurs rôles

1. **Keystone**: Authentification et autorisation
   - Flux: Valide les tokens pour chaque requête API

2. **Nova**: Calcul
   - API: Point d'entrée REST
   - Scheduler: Choisit le nœud compute
   - Compute: Gère les instances

3. **Neutron**: Réseau
   - Server: API principale
   - Agents: Implémentent le réseau (OVS, LinuxBridge)
   - DHCP: Alloue les IPs

4. **Cinder**: Stockage bloc
   - API: Gère les volumes
   - Volume: Interagit avec le backend (LVM, Ceph)

5. **Glance**: Registre d'images
   - Stocke les images de VM

## Bonnes pratiques

1. Pour Kolla-Ansible:
   - Toujours faire des `prechecks` avant le déploiement
   - Utiliser des backends séparés pour les bases de données en prod
   - Configurer le monitoring (Prometheus + Grafana)

2. Pour DevStack:
   - Utiliser des branches stables pour le développement
   - Nettoyer avec `./unstack.sh` avant de redéployer
   - Configurer des plugins spécifiques dans `local.conf`

## Dépannage initial

### Kolla-Ansible
```bash
# Voir les logs d'un service
sudo docker logs <nom_container>

# Reconfigurer après modification
sudo kolla-ansible reconfigure -i /etc/kolla/all-in-one

# Vérifier l'état des containers
sudo docker ps -a
```

### DevStack
```bash
# Logs principaux
less /opt/stack/logs/screen-*

# Redémarrer tous les services
./rejoin-stack.sh
```

## Conclusion

Cette installation vous donne une base solide pour:
- Comprendre l'architecture modulaire d'OpenStack
- Expérimenter avec les composants principaux
- Préparer un environnement de production avec Kolla-Ansible
- Développer et tester rapidement avec DevStack

Pour aller plus loin, explorez:
- L'intégration avec Ceph pour le stockage
- Les configurations multi-nœuds avec Kolla
- Les plugins réseau avancés dans Neutron


=================================================================================
# Intégration de Ceph avec OpenStack via Kolla-Ansible

## Concepts fondamentaux

### Architecture Ceph dans OpenStack

Ceph fournit un stockage distribué et hautement disponible pour plusieurs services OpenStack :

```
[Cluster Ceph]
  ├── OSD (Object Storage Daemon) - Stockage physique
  ├── MON (Monitor) - Supervision du cluster
  ├── MDS (Metadata Server) - Uniquement pour CephFS
  └── RGW (RADOS Gateway) - Interface S3/Swift
```

### Services OpenStack utilisant Ceph

1. **Glance** : Stockage des images de VM
2. **Cinder** : Volumes bloc persistants
3. **Nova** : Disques éphémères des instances
4. **Swift** : Alternative avec RGW (optionnel)

## Préparation du cluster Ceph

### 1. Installation de Ceph

Sur chaque nœud de stockage :

```bash
sudo apt install -y ceph-mon ceph-osd ceph-mgr ceph-mds ceph-common
```

### 2. Configuration initiale

```bash
sudo ceph-deploy new node1 node2 node3  # Vos nœuds Ceph
sudo ceph-deploy mon create-initial
sudo ceph-deploy admin node1 node2 node3
```

### 3. Création des pools

```bash
sudo ceph osd pool create volumes 128
sudo ceph osd pool create images 64
sudo ceph osd pool create vms 64
sudo ceph osd pool create backups 64
```

## Intégration avec Kolla-Ansible

### 1. Configuration de `/etc/kolla/globals.yml`

```yaml
enable_ceph: "yes"
enable_ceph_rgw: "no"  # Sauf si besoin de RGW
enable_ceph_mds: "no"  # Sauf si besoin de CephFS

# Configuration Ceph
ceph_glance_pool: images
ceph_cinder_pool: volumes
ceph_nova_pool: vms
ceph_cinder_backup_pool: backups

ceph_conf_overrides:
  global:
    osd_pool_default_size: 3
    osd_pool_default_min_size: 2
```

### 2. Configuration des clients Ceph

Ajoutez dans `/etc/kolla/config/ceph.conf` :

```
[client]
rbd cache = true
rbd cache writethrough until flush = true
admin socket = /var/run/ceph/guests/$cluster-$type.$id.$pid.$cctid.asok
log file = /var/log/ceph/qemu-guest-$pid.log
rbd concurrent management ops = 20
```

### 3. Déploiement avec Kolla-Ansible

```bash
sudo kolla-ansible -i /etc/kolla/multinode deploy
```

## Configuration des services OpenStack

### 1. Glance avec Ceph

```bash
openstack-config --set /etc/kolla/glance-api/glance-api.conf glance_store stores rbd
openstack-config --set /etc/kolla/glance-api/glance-api.conf glance_store default_store rbd
openstack-config --set /etc/kolla/glance-api/glance-api.conf glance_store rbd_store_pool images
openstack-config --set /etc/kolla/glance-api/glance-api.conf glance_store rbd_store_user glance
```

### 2. Cinder avec Ceph

```bash
openstack-config --set /etc/kolla/cinder-volume/cinder.conf DEFAULT enabled_backends ceph
openstack-config --set /etc/kolla/cinder-volume/cinder.conf ceph volume_driver cinder.volume.drivers.rbd.RBDDriver
openstack-config --set /etc/kolla/cinder-volume/cinder.conf ceph rbd_pool volumes
openstack-config --set /etc/kolla/cinder-volume/cinder.conf ceph rbd_ceph_conf /etc/ceph/ceph.conf
openstack-config --set /etc/kolla/cinder-volume/cinder.conf ceph rbd_user cinder
openstack-config --set /etc/kolla/cinder-volume/cinder.conf ceph rbd_flatten_volume_from_snapshot false
```

### 3. Nova avec Ceph

```bash
openstack-config --set /etc/kolla/nova-compute/nova.conf libvirt images_type rbd
openstack-config --set /etc/kolla/nova-compute/nova.conf libvirt images_rbd_pool vms
openstack-config --set /etc/kolla/nova-compute/nova.conf libvirt rbd_user nova
openstack-config --set /etc/kolla/nova-compute/nova.conf libvirt rbd_secret_uuid $(sudo cat /etc/kolla/nova-compute/nova-libvirt/rbd_secret_uuid)
```

## Création des utilisateurs Ceph

```bash
sudo ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images'
sudo ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd pool=images'
sudo ceph auth get-or-create client.nova mon 'profile rbd' osd 'profile rbd pool=vms'
sudo ceph auth get-or-create client.cinder-backup mon 'profile rbd' osd 'profile rbd pool=backups'
```

## Distribution des clés

Copiez les clés dans les containers :

```bash
sudo kolla-ansible -i /etc/kolla/multinode deploy --tags ceph-client
```

## Vérification de l'intégration

### 1. Tester Glance

```bash
openstack image create --disk-format raw --container-format bare --file cirros.img --store rbd cirros-rbd
rbd -p images ls  # Doit montrer l'image
```

### 2. Tester Cinder

```bash
openstack volume create --size 1 test-volume
rbd -p volumes ls  # Doit montrer le volume
```

### 3. Tester Nova

```bash
openstack server create --image cirros-rbd --flavor m1.tiny --network private vm1
rbd -p vms ls  # Doit montrer le disque de l'instance
```

## Bonnes pratiques

1. **Dimensionnement** :
   - Au moins 3 nœuds Ceph pour la redondance
   - Pools séparés pour chaque service
   - OSDs sur disques dédiés (pas de système de fichiers partagé)

2. **Performances** :
   - Utiliser des SSD pour les journals Ceph
   - Ajuster le nombre de PGs selon la taille du cluster
   ```bash
   ceph osd pool set volumes pg_num 128
   ceph osd pool set volumes pgp_num 128
   ```

3. **Monitoring** :
   - Activer le dashboard Ceph :
   ```bash
   ceph mgr module enable dashboard
   ceph dashboard create-self-signed-cert
   ```

## Dépannage courant

### Problème d'authentification

```bash
# Vérifier les permissions
sudo ceph auth get client.glance

# Regénérer si nécessaire
sudo ceph auth del client.glance
sudo ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images'
```

### Problème de performance

1. Vérifier l'état du cluster :
```bash
ceph osd perf
ceph osd df
```

2. Optimiser la configuration :
```yaml
ceph_conf_overrides:
  osd:
    filestore max sync interval: 5
    filestore min sync interval: 1
    journal max write bytes: 1073714824
    journal max write entries: 10000
```

## Architecture avancée

Pour une installation de production :

```
[OpenStack Nodes]
  ├── Controller Nodes (3+) 
  │    ├── API Services
  │    └── Ceph MON
  │
  ├── Compute Nodes
  │    ├── Nova Compute
  │    └── Ceph OSD (optionnel)
  │
  └── Storage Nodes (3+)
       ├── Ceph OSD (10+ par nœud)
       └── Ceph MON/MGR
```

Cette intégration permet d'obtenir :
- Une haute disponibilité du stockage
- Des performances scalable
- Une simplification de l'architecture (stockage unifié)
- Des fonctionnalités avancées (snapshots, clones)

=============================================================================

# Configuration Multi-nœuds avec Kolla-Ansible

## Architecture de Référence pour un Déploiement Production

### Topologie Recommandée (3 nœuds minimum)

```
[3+ Nœuds Contrôleurs]
  ├── Services API (Nova, Neutron, Glance, etc.)
  ├── Base de données (Galera Cluster)
  ├── Message Queue (RabbitMQ Cluster)
  ├── HAProxy + Keepalived (VIP)
  └── Ceph MON (optionnel)

[2+ Nœuds Compute]
  ├── Nova Compute
  └── Ceph OSD (si stockage hyperconvergé)

[3+ Nœuds Stockage]
  ├── Ceph OSD (10+ par nœud)
  └── Ceph MON/MGR (si dédié)
```

## Configuration de l'Inventaire Multi-nœuds

### Fichier `/etc/kolla/multinode`

```ini
[control]
# Premier nœud contrôleur (peut aussi héberger des services spéciaux)
controller1 ansible_user=root

# Autres nœuds contrôleurs
controller2 ansible_user=root
controller3 ansible_user=root

[network]
# Nœuds réseau (optionnel - pour déporter les agents Neutron)
network1 ansible_user=root
network2 ansible_user=root

[compute]
# Nœuds de calcul
compute1 ansible_user=root
compute2 ansible_user=root

[monitoring]
# Nœuds de monitoring (optionnel)
controller1  # Peut être colocalisé

[storage]
# Nœuds de stockage Ceph
storage1 ansible_user=root
storage2 ansible_user=root
storage3 ansible_user=root

[deployment]
# Nœud depuis lequel vous déployez
deploy-node ansible_user=root

# Groupes spéciaux
[haproxy:children]
control

[rabbitmq:children]
control

[mariadb:children]
control
```

## Configuration des Variables Globales

### Modifications clés dans `/etc/kolla/globals.yml`

```yaml
# Configuration réseau
network_interface: "eth0"
neutron_external_interface: "eth1"
kolla_internal_vip_address: "192.168.1.100"
keepalived_virtual_router_id: "51"

# Haute Disponibilité
enable_haproxy: "yes"
enable_keepalived: "yes"
enable_mariadb: "yes"
enable_rabbitmq: "yes"

# Configuration des services
nova_compute_virt_type: "kvm"
neutron_plugin_agent: "openvswitch"

# Si utilisation de Ceph (recommandé)
enable_ceph: "yes"
enable_ceph_rgw: "no"
enable_ceph_mds: "no"
```

## Préparation des Nœuds

### Sur tous les nœuds :

1. **Configuration réseau** :
   - IP statiques
   - Résolution DNS/hostname correcte
   - Connectivité entre tous les nœuds

2. **Prérequis système** :
```bash
# Sur Ubuntu/Debian
sudo apt update
sudo apt install -y python3 docker.io
```

3. **Configuration Docker** :
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/kolla.conf << EOF
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --log-opt max-size=50m --log-opt max-file=5
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## Déploiement en Multi-nœuds

### 1. Bootstrap initial

```bash
sudo kolla-ansible -i /etc/kolla/multinode bootstrap-servers
```

### 2. Vérification des prérequis

```bash
sudo kolla-ansible -i /etc/kolla/multinode prechecks
```

### 3. Déploiement principal

```bash
sudo kolla-ansible -i /etc/kolla/multinode deploy
```

### 4. Configuration post-déploiement

```bash
sudo kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
```

## Configuration Avancée

### 1. Haute Disponibilité pour les Services

Dans `/etc/kolla/globals.yml` :

```yaml
# MariaDB Galera Cluster
mariadb_max_connections: 1000
galera_cluster_name: "openstack_galera"
galera_wsrep_cluster_address: "gcomm://controller1,controller2,controller3"

# RabbitMQ Cluster
rabbitmq_cluster_name: "openstack"
rabbitmq_cluster_nodes: "rabbit@controller1, rabbit@controller2, rabbit@controller3"
```

### 2. Configuration Neutron Multi-nœuds

Pour les nœuds réseau :

```yaml
# Dans globals.yml
neutron_external_interface: "eth1"
enable_neutron_dvr: "yes"
enable_neutron_metadata: "yes"
neutron_plugin_agent: "openvswitch"
```

### 3. Nova Multi-nœuds

Configuration des cellules (pour très grands déploiements) :

```bash
# Après le déploiement initial
openstack cell_v2 map_cell0 --database_connection mysql+pymysql://nova:password@controller1/nova_cell0
openstack cell_v2 create_cell --name=cell1 --verbose
```

## Gestion des Nœuds Compute

### Ajout d'un nouveau nœud compute

1. Ajouter le nœud dans `/etc/kolla/multinode` sous `[compute]`
2. Exécuter :

```bash
sudo kolla-ansible -i /etc/kolla/multinode bootstrap-servers --limit compute_new_node
sudo kolla-ansible -i /etc/kolla/multinode deploy --limit compute_new_node
```

### Retrait d'un nœud compute

1. Mettre le nœud en maintenance :
```bash
openstack compute service set --disable <hostname> nova-compute
```

2. Migrer les instances :
```bash
openstack server migrate <instance-id>
```

3. Retirer le nœud de l'inventaire

## Surveillance et Maintenance

### Vérification de l'état du cluster

```bash
# Vérifier l'état des services
openstack compute service list
openstack network agent list

# Vérifier la santé Ceph (si utilisé)
sudo docker exec ceph_mon ceph -s

# Vérifier les clusters MariaDB/RabbitMQ
sudo docker exec mariadb mysql -e "SHOW STATUS LIKE 'wsrep%';"
sudo docker exec rabbitmq rabbitmqctl cluster_status
```

### Mises à jour

Procédure de mise à jour sécurisée :

```bash
# 1. Sauvegarder les configurations
sudo cp -a /etc/kolla /etc/kolla.bak

# 2. Mettre à jour les containers
sudo kolla-ansible -i /etc/kolla/multinode pull

# 3. Reconfigurer
sudo kolla-ansible -i /etc/kolla/multinode reconfigure

# 4. Redémarrer si nécessaire
sudo kolla-ansible -i /etc/kolla/multinode restart
```

## Bonnes Pratiques pour les Déploiements Multi-nœuds

1. **Séparation des rôles** :
   - Garder les contrôleurs dédiés aux services API
   - Éviter de mélanger contrôleurs et compute nodes

2. **Réseau** :
   - Au moins 2 interfaces réseau (management + données)
   - VLANs séparés pour le trafic de gestion, stockage, etc.

3. **Stockage** :
   - Pour les petits déploiements : hyperconvergé (Ceph sur compute nodes)
   - Pour les grands déploiements : stockage dédié

4. **Monitoring** :
   - Déployer Prometheus/Grafana
   - Surveiller particulièrement :
     - Charge des contrôleurs
     - Espace disque Ceph
     - État des clusters (Galera, RabbitMQ)

## Dépannage Multi-nœuds

### Problèmes courants et solutions

1. **Problèmes de synchronisation MariaDB** :
```bash
# Sur un nœud contrôleur
sudo docker exec -it mariadb mysql -e "SHOW STATUS LIKE 'wsrep%';"
# Si un nœud est désynchronisé
sudo docker restart mariadb
```

2. **Problèmes RabbitMQ** :
```bash
sudo docker exec -it rabbitmq rabbitmqctl cluster_status
# Pour rejoindre un nœud au cluster
sudo docker exec rabbitmq rabbitmqctl stop_app
sudo docker exec rabbitmq rabbitmqctl join_cluster rabbit@controller1
sudo docker exec rabbitmq rabbitmqctl start_app
```

3. **Problèmes réseau entre nœuds** :
   - Vérifier les pare-feux
   - Tester la connectivité sur tous les ports nécessaires

## Scaling Horizontal

### Ajout de contrôleurs supplémentaires

1. Ajouter le nouveau nœud à `[control]` dans l'inventaire
2. Exécuter :
```bash
sudo kolla-ansible -i /etc/kolla/multinode bootstrap-servers --limit new_controller
sudo kolla-ansible -i /etc/kolla/multinode deploy --limit new_controller
```

### Configuration des Zones de Disponibilité

1. Créer des host aggregates :
```bash
openstack aggregate create --zone az1 az1
openstack aggregate add host az1 compute1
```

2. Lors du lancement d'instances :
```bash
openstack server create --image cirros --flavor m1.tiny --network private --availability-zone az1 vm1
```

Cette configuration multi-nœuds avec Kolla-Ansible fournit une plateforme OpenStack hautement disponible, scalable et adaptée aux environnements de production.

=============================================================================

# Plugins Réseaux Avancés dans Neutron

## Architecture Fondamentale de Neutron

Neutron utilise une architecture modulaire avec :
- **Core plugin** : Gère les réseaux, sous-réseaux et ports de base
- **Service plugins** : Ajoute des fonctionnalités avancées (LBaaS, FWaaS, VPNaaS)
- **Agents** : Implémentent la connectivité réseau sur les nœuds

```
[API Neutron]
  ├── Core Plugin (ML2)
  │    ├── Type Drivers (local, flat, vlan, vxlan, gre)
  │    └── Mechanism Drivers (openvswitch, linuxbridge, sriov, etc.)
  │
  ├── Service Plugins
  │    ├── Router (L3)
  │    ├── Load Balancer (LBaaS)
  │    ├── Firewall (FWaaS)
  │    └── VPN (VPNaaS)
  │
  └── Agents
       ├── L2 (OVS, LinuxBridge)
       ├── L3 (Router)
       ├── DHCP
       └── Métadonnées
```

## Plugins de Couche 2 Avancés

### 1. Open vSwitch (OVS) avec VXLAN/GRE

**Configuration dans `/etc/kolla/neutron-server/ml2_conf.ini`** :

```ini
[ml2]
type_drivers = flat,vlan,vxlan,gre
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population

[ml2_type_vxlan]
vni_ranges = 1000:2000

[ovs]
bridge_mappings = physnet1:br-ex
local_ip = <IP_Interne_Du_Nœud>
enable_tunneling = true
```

**Cas d'utilisation** :
- Réseaux overlay multi-locataires
- Isolation de trafic sans nécessité de VLANs physiques

### 2. SR-IOV (Single Root I/O Virtualization)

**Configuration** :

```ini
[ml2]
mechanism_drivers = openvswitch,sriovnicswitch

[ml2_sriov]
supported_pci_vendor_devs = 8086:10ed,15b3:1004

[sriov_nic]
physical_device_mappings = physnet2:ens2f0
```

**Prérequis** :
- Activation du SR-IOV dans le BIOS
- Drivers PF/VF installés
- Configuration des interfaces PF :

```bash
echo 4 > /sys/class/net/ens2f0/device/sriov_numvfs
```

**Avantages** :
- Performance quasi-native (bypass l'hyperviseur)
- Latence réduite pour les workloads sensibles

## Plugins de Couche 3 Avancés

### 1. Distributed Virtual Routing (DVR)

**Activation dans `/etc/kolla/globals.yml`** :

```yaml
enable_neutron_dvr: "yes"
```

**Configuration additionnelle** :

```ini
[agent]
enable_distributed_routing = True

[default]
router_distributed = True
```

**Flux de trafic** :
- Trafic Est-Ouest : Routé localement sur le nœud compute
- Trafic Nord-Sud : Sortie via les nœuds réseau spéciaux

### 2. BGP Dynamic Routing (BGP DR)

Pour l'intégration avec les routeurs physiques :

```ini
[bgp]
bgp_speakers = 192.168.1.1:64512
local_as = 64513
advertise_floating_ip_host_routes = true
advertise_tenant_networks = true
```

## Plugins de Services Avancés

### 1. Load Balancing as a Service (LBaaS)

**Avec Octavia (recommandé)** :

```yaml
# Dans globals.yml
enable_octavia: "yes"
octavia_network_interface: "eth0"
```

**Architecture Octavia** :
- **Amphorae** : Instances dédiées faisant le load balancing
- **Controller** : Gère le cycle de vie des amphorae

**Exemple de création** :

```bash
openstack loadbalancer create --name web-lb --vip-subnet-id private-subnet
openstack loadbalancer listener create --name http-listener --protocol HTTP --protocol-port 80 web-lb
openstack loadbalancer pool create --name http-pool --lb-algorithm ROUND_ROBIN --listener http-listener --protocol HTTP
openstack loadbalancer member create --subnet-id private-subnet --address 192.168.1.10 --protocol-port 80 http-pool
```

### 2. Firewall as a Service (FWaaS)

**Version v2 avec groupes de sécurité** :

```yaml
# Activation dans globals.yml
enable_neutron_fwaas: "yes"
```

**Exemple de configuration** :

```bash
# Créer une politique
openstack firewall group policy create --firewall-rule allow-web allow-web-policy

# Ajouter des règles
openstack firewall group rule create --name allow-http --protocol tcp --destination-port 80 --action allow
openstack firewall group rule create --name allow-https --protocol tcp --destination-port 443 --action allow

# Appliquer au groupe
openstack firewall group policy set --firewall-rule allow-http allow-web-policy
openstack firewall group policy set --firewall-rule allow-https allow-web-policy

# Créer le firewall
openstack firewall group create --name web-fw --ingress-firewall-policy allow-web-policy --egress-firewall-policy allow-web-policy
```

## Intégrations Avancées

### 1. NSX-T Integration

Pour les environnements hybrides VMware/OpenStack :

```ini
[ml2]
mechanism_drivers = nsxv3

[nsxv3]
nsx_manager = https://nsx-manager-ip
nsx_username = admin
nsx_password = secret
t0_router = t0-router-id
```

### 2. OVN (Open Virtual Network)

Alternative moderne à OVS :

```yaml
# Dans globals.yml
enable_ovn: "yes"
neutron_plugin_agent: "ovn"
```

**Avantages** :
- Architecture simplifiée
- Meilleure évolutivité
- Intégration native avec Kubernetes

## Optimisation des Performances

### 1. Configuration DPDK (Data Plane Development Kit)

Pour les workloads réseau intensifs :

```ini
[ovs]
datapath_type = netdev
vhostuser_socket_dir = /var/run/openvswitch
```

**Prérequis** :
- CPUs isolés (via kernel parameters)
- Hugepages configurés
- Interfaces liées à DPDK :

```bash
dpdk-devbind.py --bind=vfio-pci ens2f0
```

### 2. Accélération Hardware (SmartNICs)

Avec des cartes comme Mellanox ConnectX ou Intel FPGA :

```ini
[ml2_mechanism_openvswitch]
supported_vnic_types = normal,direct,direct-physical,macvtap
```

## Bonnes Pratiques de Configuration

1. **Séparation des plans de données** :
   - VLAN dédié pour le trafic de gestion
   - VLAN séparé pour les tunnels VXLAN/GRE
   - Interface dédiée pour le trafic public

2. **Sécurité** :
   - Activer les groupes de sécurité par défaut
   - Limiter l'accès à l'API Neutron
   - Utiliser TLS pour toutes les communications

3. **Monitoring** :
   - Surveiller l'utilisation des IPs/floating IPs
   - Tracer les performances des agents L2/L3
   - Alertes sur les échecs de tunnels

## Dépannage Avancé

### Outils clés :

1. **OVS Commands** :
```bash
ovs-vsctl show
ovs-ofctl dump-flows br-int
```

2. **Espace Noms Linux** :
```bash
ip netns list
ip netns exec qrouter-XXXX ping 8.8.8.8
```

3. **Logs Neutron** :
```bash
docker logs neutron_server
docker logs neutron_openvswitch_agent
```

### Problèmes courants :

1. **Tunnels VXLAN qui tombent** :
   - Vérifier `local_ip` correcte
   - Vérifier MTU (doit être <= MTU physique - overhead)

2. **Problèmes de performance DVR** :
   - Activer le conntrack caching :
   ```ini
   [agent]
   conntrack_helpers_timeout = 120
   ```

3. **Échecs SR-IOV** :
   - Vérifier que les VFs sont créés
   - Confirmer que le driver VF est chargé
   ```bash
   lspci -k | grep -i ethernet
   ```

Ces plugins avancés permettent d'adapter Neutron à des cas d'utilisation complexes tout en maintenant des performances optimales. Le choix dépendra des besoins spécifiques en termes de performance, isolation et intégration avec l'infrastructure existante.