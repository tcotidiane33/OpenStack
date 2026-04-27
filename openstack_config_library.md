# 📂 Bibliothèque de Configurations OpenStack

Ce document regroupe les fichiers de configuration essentiels pour OpenStack, classés par cas d'usage et scénario de déploiement.

---

## 1. DevStack (`local.conf`)
Fichier à placer dans `/opt/stack/devstack/local.conf`.

### 🔹 Cas 1 : Minimaliste (Apprentissage rapide)
Idéal pour les machines avec peu de RAM (8 Go).
```ini
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# Services minimum pour Nova/Neutron/Glance/Keystone
ENABLED_SERVICES=rabbit,mysql,key,n-api,n-cond,n-sch,n-cpu,n-novnc,n-api-meta,g-api,q-svc,q-agt,q-dhcp,q-l3,q-meta,horizon
```

### 🔹 Cas 2 : Full Stack (Lab Complet)
Inclut Cinder (Stockage), Heat (Orchestration) et Swift (Objet).
```ini
[[local|localrc]]
ADMIN_PASSWORD=secret
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

# Activer tout le stack standard
ENABLED_SERVICES+=,c-api,c-vol,c-bak,heat,h-api,h-api-cfn,h-api-cw,h-eng,s-proxy,s-obj,s-container,s-account

# Configuration réseau spécifique
HOST_IP=192.168.1.10
FLOATING_RANGE=192.168.1.224/27
```

---

## 2. Kolla-Ansible (`globals.yml`)
Fichier à placer dans `/etc/kolla/globals.yml`.

### 🔹 Cas 1 : All-in-One (AIO) - Docker Simple
```yaml
---
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "zed"
kolla_internal_vip_address: "10.0.0.100"
network_interface: "eth0"
neutron_external_interface: "eth1"
enable_haproxy: "no"
```

### 🔹 Cas 2 : Haute Disponibilité (HA) - Multi-nœuds
```yaml
---
kolla_internal_vip_address: "192.168.1.200"
enable_haproxy: "yes"
enable_keepalived: "yes"
enable_mariadb: "yes"
enable_rabbitmq: "yes"

# Configuration multi-interfaces
network_interface: "eth0"          # Management
kolla_external_vip_address: "1.2.3.4" # Accès public API
neutron_external_interface: "eth1" # Trafic VMs
```

### 🔹 Cas 3 : Avec Stockage Ceph Intégré
```yaml
---
enable_ceph: "yes"
enable_ceph_rgw: "yes"
ceph_glance_pool_name: "images"
ceph_cinder_pool_name: "volumes"
ceph_nova_pool_name: "vms"
```

---

## 3. Orchestration Heat (`template.yaml`)

### 🔹 Cas 1 : Serveur Web + Floating IP
```yaml
heat_template_version: 2018-08-31
description: Déploiement d'une instance web avec IP publique.

parameters:
  key_name:
    type: string
    default: mykey
  image:
    type: string
    default: ubuntu-22.04

resources:
  my_instance:
    type: OS::Nova::Server
    properties:
      key_name: { get_param: key_name }
      image: { get_param: image }
      flavor: m1.small
      networks: [{ network: private_net }]

  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: public_net

  association:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: { get_resource: floating_ip }
      port_id: { get_attr: [my_instance, addresses, private_net, 0, port] }
```

---

## 4. Initialisation Cloud-init (`user-data.yml`)
À passer via `--user-data` lors du `openstack server create`.

### 🔹 Cas 1 : Serveur Web Auto-installé (Nginx)
```yaml
#cloud-config
package_update: true
packages:
  - nginx
runcmd:
  - systemctl start nginx
  - echo "<h1>Bienvenue sur mon Cloud OpenStack</h1>" > /var/www/html/index.html
```

---

## 5. Configuration Réseau Neutron (ML2/OVS)
Exemple de configuration `ml2_conf.ini` pour de la production.

```ini
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population

[ml2_type_vxlan]
vni_ranges = 1:1000

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```
