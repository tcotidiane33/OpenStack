# 🎓 BOOTCAMP OPENSTACK — JOUR 4 : Avancé, Orchestration & Dépannage

---

## 🎯 Objectifs du Jour
- Automatiser des déploiements avec Heat (Orchestration)
- Comprendre et pratiquer la méthodologie de dépannage
- Maîtriser la supervision des services
- Connaître les bonnes pratiques de production

---

## ⏱️ Planning du Jour

| Horaire | Session | Type |
|---------|---------|------|
| 09h00 - 10h30 | Module 1 : Orchestration avec Heat | Théorie |
| 10h30 - 11h00 | ☕ Pause | — |
| 11h00 - 12h30 | Module 2 : Méthodologie de Dépannage | Théorie |
| 12h30 - 13h30 | 🍽️ Déjeuner | — |
| 13h30 - 15h30 | 🔧 Atelier 4A : Template Heat | Pratique |
| 15h30 - 17h30 | 🔧 Atelier 4B : Dépannage Simulé | Pratique |

---

# MODULE 1 — Orchestration avec Heat

## 1.1 Pourquoi l'Orchestration ?

> **Analogie** : Créer votre infrastructure manuellement, c'est comme construire une maison brique par brique à la main.
> Heat, c'est l'architecte qui lit les **plans** (templates) et construit tout automatiquement, dans le bon ordre, en une seule commande.

### Le Problème sans Orchestration

```
SANS HEAT (manuel) :
1. Créer le réseau
2. Créer le sous-réseau
3. Créer le routeur
4. Connecter le routeur
5. Créer le security group
6. Ajouter les règles (x5)
7. Créer la keypair
8. Lancer la VM
9. Créer le volume
10. Attacher le volume
11. Allouer la floating IP
12. Associer la floating IP
→ 12 commandes, 12 points de défaillance potentiels
   Reproductible ? Difficile. Versionnable ? Pas du tout.
```

```
AVEC HEAT :
openstack stack create -t mon-infra.yaml ma-stack
→ 1 commande. Toujours identique. Versionnable dans Git. ✅
```

## 1.2 La Structure d'un Template Heat (HOT)

HOT = **H**eat **O**rchestration **T**emplate (format YAML)

```yaml
heat_template_version: 2021-04-16  # Version du format

description: >
  Description de ce que fait ce template

# ─── Paramètres (inputs) ──────────────────────────────
parameters:
  nom_du_parametre:
    type: string | number | boolean | json | comma_delimited_list
    default: valeur_par_defaut
    description: Description du paramètre
    constraints:
      - allowed_values: [valeur1, valeur2]

# ─── Ressources (le cœur du template) ────────────────
resources:
  nom_logique_ressource:
    type: OS::Service::TypeRessource
    properties:
      propriete1: valeur
      propriete2: { get_param: nom_du_parametre }  # Référence un paramètre

# ─── Sorties (outputs) ────────────────────────────────
outputs:
  nom_output:
    description: Description de la sortie
    value: { get_attr: [nom_ressource, attribut] }
```

## 1.3 Les Types de Ressources Principaux

| Type Heat | Équivalent CLI | Description |
|-----------|----------------|-------------|
| `OS::Nova::Server` | `openstack server create` | Créer une VM |
| `OS::Nova::KeyPair` | `openstack keypair create` | Paire de clés SSH |
| `OS::Neutron::Net` | `openstack network create` | Réseau virtuel |
| `OS::Neutron::Subnet` | `openstack subnet create` | Sous-réseau |
| `OS::Neutron::Router` | `openstack router create` | Routeur virtuel |
| `OS::Neutron::FloatingIP` | `openstack floating ip create` | IP flottante |
| `OS::Neutron::SecurityGroup` | `openstack security group create` | Groupe de sécurité |
| `OS::Cinder::Volume` | `openstack volume create` | Volume bloc |
| `OS::Cinder::VolumeAttachment` | `openstack server add volume` | Attacher un volume |

## 1.4 Les Fonctions Intrinsèques

```yaml
# Référencer un paramètre
image: { get_param: server_image }

# Référencer un attribut d'une autre ressource
network: { get_resource: mon_reseau }

# Récupérer un attribut après création
value: { get_attr: [mon_serveur, first_address] }

# Concaténer des strings
name: { str_replace: { template: "serveur-SUFFIX", params: { SUFFIX: { get_param: env } } } }

# Valeur conditionnelle
if: [condition_name, true_value, false_value]
```

---

# MODULE 2 — Méthodologie de Dépannage

## 2.1 L'Approche Structurée "Top-Down"

```
SYMPTÔME : "Ma VM ne démarre pas" ou "Je n'ai pas accès"
      │
      ▼
1. KEYSTONE OK ?     → openstack token issue
      │
      ▼
2. NOVA OK ?         → openstack compute service list
      │
      ▼
3. GLANCE OK ?       → openstack image list
      │
      ▼
4. NEUTRON OK ?      → openstack network agent list
      │
      ▼
5. CINDER OK ? (si volume) → openstack volume service list
      │
      ▼
6. LOGS DU SERVICE EN ERREUR
```

## 2.2 Localisation des Logs

### Avec Kolla-Ansible (Docker)
```bash
# Voir les containers actifs
docker ps

# Logs d'un service spécifique
docker logs nova_api --tail 100
docker logs nova_compute --tail 100
docker logs neutron_server --tail 100
docker logs keystone --tail 100
docker logs glance_api --tail 100
docker logs cinder_volume --tail 100
docker logs mariadb --tail 100
docker logs rabbitmq --tail 50

# Logs en temps réel
docker logs -f nova_compute

# Exécuter une commande dans un container
docker exec -it nova_api bash
```

### Avec DevStack (Systemd/Screen)
```bash
# Voir les logs via journalctl
journalctl -u devstack@n-api -f
journalctl -u devstack@n-cpu -f
journalctl -u devstack@q-svc -f
journalctl -u devstack@keystone -f

# Fichiers de logs directs
tail -f /opt/stack/logs/n-api.log
tail -f /opt/stack/logs/q-svc.log
tail -f /var/log/nova/nova-compute.log
```

## 2.3 Tableau des Erreurs Fréquentes

| Symptôme | Commande de diagnostic | Cause probable |
|----------|----------------------|----------------|
| VM en état ERROR | `openstack server show <id>` | Pas assez de ressources compute |
| "No valid host was found" | `openstack compute service list` | Nova-scheduler ne trouve pas de nœud |
| VM stuck en BUILD | `docker logs nova_compute` | Problème KVM/image |
| Réseau inaccessible | `openstack network agent list` | Agent Neutron down |
| Floating IP ne marche pas | `openstack router show <r>` | Pas de gateway sur le routeur |
| Volume non attaché | `openstack volume show <v>` | Cinder-volume down |
| Login Horizon échoue | `docker logs keystone` | Keystone erreur DB |

## 2.4 Commandes de Diagnostic Avancées

```bash
# ─── Nova ────────────────────────────────────────────
# Voir les ressources disponibles sur chaque hyperviseur
openstack hypervisor stats show
openstack hypervisor list --long

# Voir l'allocation des ressources
openstack host show <nom-du-compute>

# ─── Neutron ──────────────────────────────────────────
# Vérifier les agents en détail
openstack network agent list --long

# Vérifier la connectivité L2
openstack port list --server <id-vm>

# Voir les namespaces réseau (sur le nœud réseau)
ip netns list
# → qrouter-XXXXX, qdhcp-XXXXX

# Tester la connectivité dans un namespace
ip netns exec qrouter-XXXXX ping 8.8.8.8

# ─── RabbitMQ ─────────────────────────────────────────
# Vérifier les queues (communication inter-services)
docker exec rabbitmq rabbitmqctl list_queues
docker exec rabbitmq rabbitmqctl cluster_status

# ─── MariaDB ──────────────────────────────────────────
docker exec mariadb mysql -u root -e "SHOW DATABASES;"
docker exec mariadb mysql -u root -e "SHOW STATUS LIKE 'wsrep%';"
```

---

# 🔧 ATELIER 4A — Template Heat : Infrastructure Automatisée

## Mission : Créer un Template qui Déploie un Serveur Web en 1 Commande

### Template complet : `web-stack.yaml`

```bash
cat > web-stack.yaml << 'EOF'
heat_template_version: 2021-04-16

description: >
  Stack complète : réseau + VM + floating IP + volume.
  Usage: openstack stack create -t web-stack.yaml -e env.yaml ma-stack

parameters:
  key_name:
    type: string
    description: Nom de la paire de clés SSH
    default: bootcamp-key

  image_name:
    type: string
    description: Nom de l'image à utiliser
    default: Cirros-0.6.2

  flavor:
    type: string
    description: Gabarit de la VM
    default: m1.small

  external_network:
    type: string
    description: Nom du réseau externe (pour les floating IPs)
    default: external-net

  dns_server:
    type: string
    description: Serveur DNS
    default: 8.8.8.8

  subnet_cidr:
    type: string
    description: CIDR du réseau interne
    default: 10.20.30.0/24

  volume_size:
    type: number
    description: Taille du volume en GB
    default: 10

resources:

  # ─── Réseau ──────────────────────────────────────────
  private_net:
    type: OS::Neutron::Net
    properties:
      name:
        str_replace:
          template: "stack-net-STACKNAME"
          params:
            STACKNAME: { get_param: "OS::stack_name" }

  private_subnet:
    type: OS::Neutron::Subnet
    properties:
      network: { get_resource: private_net }
      cidr: { get_param: subnet_cidr }
      dns_nameservers: [{ get_param: dns_server }]

  # ─── Routeur ──────────────────────────────────────────
  router:
    type: OS::Neutron::Router
    properties:
      external_gateway_info:
        network: { get_param: external_network }

  router_interface:
    type: OS::Neutron::RouterInterface
    properties:
      router: { get_resource: router }
      subnet: { get_resource: private_subnet }

  # ─── Security Group ──────────────────────────────────
  web_sg:
    type: OS::Neutron::SecurityGroup
    properties:
      name:
        str_replace:
          template: "stack-sg-STACKNAME"
          params:
            STACKNAME: { get_param: "OS::stack_name" }
      rules:
        - protocol: icmp
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
        - protocol: tcp
          port_range_min: 80
          port_range_max: 80

  # ─── Instance ──────────────────────────────────────────
  web_server:
    type: OS::Nova::Server
    depends_on: router_interface
    properties:
      name:
        str_replace:
          template: "web-server-STACKNAME"
          params:
            STACKNAME: { get_param: "OS::stack_name" }
      image: { get_param: image_name }
      flavor: { get_param: flavor }
      key_name: { get_param: key_name }
      networks:
        - network: { get_resource: private_net }
      security_groups:
        - { get_resource: web_sg }
      user_data_format: RAW
      user_data: |
        #!/bin/bash
        # Script d'initialisation cloud-init
        echo "=== OpenStack Bootcamp ===" > /tmp/boot.log
        echo "Serveur démarré le $(date)" >> /tmp/boot.log
        # Installer un serveur web simple si disponible
        which python3 && python3 -m http.server 80 & || true

  # ─── Volume ──────────────────────────────────────────
  data_volume:
    type: OS::Cinder::Volume
    properties:
      size: { get_param: volume_size }
      name:
        str_replace:
          template: "vol-data-STACKNAME"
          params:
            STACKNAME: { get_param: "OS::stack_name" }

  volume_attachment:
    type: OS::Cinder::VolumeAttachment
    properties:
      volume_id: { get_resource: data_volume }
      instance_uuid: { get_resource: web_server }

  # ─── Floating IP ──────────────────────────────────────
  floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: { get_param: external_network }

  floating_ip_assoc:
    type: OS::Neutron::FloatingIPAssociation
    properties:
      floatingip_id: { get_resource: floating_ip }
      port_id: { get_attr: [web_server, addresses, { get_resource: private_net }, 0, port] }

outputs:
  server_ip_private:
    description: IP privée du serveur
    value: { get_attr: [web_server, first_address] }

  server_ip_public:
    description: Floating IP publique du serveur
    value: { get_attr: [floating_ip, floating_ip_address] }

  ssh_command:
    description: Commande SSH pour se connecter
    value:
      str_replace:
        template: "ssh -i ~/.ssh/openstack-bootcamp cirros@FLOATINGIP"
        params:
          FLOATINGIP: { get_attr: [floating_ip, floating_ip_address] }
EOF

echo "✅ Template Heat créé : web-stack.yaml"
```

### Déployer la Stack

```bash
# Valider le template sans déployer
openstack orchestration template validate -t web-stack.yaml

# Déployer la stack
openstack stack create \
  -t web-stack.yaml \
  --wait \
  ma-web-stack

# Suivre la progression
openstack stack event list ma-web-stack --follow

# Voir les outputs
openstack stack output show --all ma-web-stack

# Voir les ressources créées
openstack stack resource list ma-web-stack
```

### Mettre à Jour la Stack

```bash
# Modifier le template (ex: augmenter le volume)
sed -i 's/default: 10/default: 20/' web-stack.yaml

# Mettre à jour
openstack stack update -t web-stack.yaml --wait ma-web-stack
```

### Supprimer la Stack (Cleanup)

```bash
# Supprime TOUTES les ressources de la stack en cascade
openstack stack delete --wait --yes ma-web-stack

# Vérifier que tout est bien supprimé
openstack server list
openstack network list
openstack floating ip list
```

---

# 🔧 ATELIER 4B — Dépannage Simulé

## Scénario de Panne 1 : "Ma VM ne démarre plus"

```bash
# Simuler une panne
openstack server set --state error ma-vm

# Diagnostiquer
openstack server show ma-vm -c status -c fault
openstack server show ma-vm

# Lire les logs
openstack console log show ma-vm

# Solution : Reset de l'état
openstack server set --state active ma-vm

# Ou hard reboot
openstack server reboot --hard ma-vm
```

## Scénario de Panne 2 : "Le réseau de ma VM ne fonctionne plus"

```bash
# Vérifier les agents
openstack network agent list

# Si un agent est down, identifier le service à redémarrer
# Pour Kolla :
docker restart neutron_openvswitch_agent
# ou
docker restart neutron_l3_agent

# Vérifier les ports de la VM
openstack port list --server ma-vm
openstack port show <port-id>

# Vérifier le security group
openstack server show ma-vm -c security_groups
openstack security group rule list <sg-id>
```

## Scénario de Panne 3 : "Je ne peux pas créer de nouvelles VMs"

```bash
# Vérifier les quotas
openstack quota show  # Projet actuel
openstack quota show --detail  # Avec utilisation actuelle

# Vérifier les ressources disponibles
openstack hypervisor stats show
openstack hypervisor list --long

# Si quota atteint (admin seulement)
openstack quota set --instances 50 mon-projet

# Si ressources physiques insuffisantes
# → Il faut ajouter un nœud compute
```

---

# 📋 BONNES PRATIQUES PRODUCTION

## Sécurité

```bash
# 1. Ne jamais utiliser le projet "admin" pour les workloads
#    → Créer des projets dédiés

# 2. Changer les mots de passe par défaut
openstack user set --password "NouveauMotDePasse!" admin

# 3. Activer TLS sur les endpoints
# Dans /etc/kolla/globals.yml :
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"

# 4. Auditer les accès régulièrement
openstack role assignment list --names

# 5. Utiliser des tokens à durée de vie courte
# Dans keystone.conf :
# expiration = 3600  (1 heure par défaut)
```

## Surveillance

```bash
# Commandes de surveillance quotidiennes
alias os-health='echo "=== COMPUTE ===" && openstack compute service list && \
  echo "=== NETWORK ===" && openstack network agent list && \
  echo "=== VOLUME ===" && openstack volume service list'

# Ajouter à un script cron
os-health
```

## Sauvegarde

```bash
# Snapshot de toutes les VMs d'un projet
for vm in $(openstack server list -f value -c ID); do
  name=$(openstack server show $vm -f value -c name)
  echo "Snapshot de $name..."
  openstack server image create \
    --name "backup-${name}-$(date +%Y%m%d)" \
    $vm
done

# Snapshot de tous les volumes
for vol in $(openstack volume list -f value -c ID); do
  name=$(openstack volume show $vol -f value -c name)
  openstack volume snapshot create \
    --volume $vol \
    --name "backup-${name}-$(date +%Y%m%d)" \
    --force
done
```

---

## 🏆 PROJET FINAL DU BOOTCAMP

### Mission : Déployer une Architecture 3-Tiers

Créez un template Heat qui déploie :

```
                    INTERNET
                        │
                [Load Balancer / Bastion]
               /           |           \
       [Web 1]         [Web 2]        [Web 3]
           \               |              /
                    [Base de données]
                           |
                    [Volume de données]
```

**Ressources à créer :**
1. 3 VMs web (flavors `m1.tiny` + image Cirros)
2. 1 VM DB (`m1.small`)
3. 1 Volume pour la DB (20 GB)
4. 1 Floating IP sur le bastion
5. Security groups appropriés :
   - `sg-bastion` : SSH depuis partout
   - `sg-web` : HTTP depuis partout, SSH depuis bastion seulement
   - `sg-db` : Port 3306 depuis sg-web seulement
6. Tous les réseaux et routeurs nécessaires

---

## 📚 Résumé du Bootcamp Complet

```
JOUR 1 : Fondations
  ✅ Architecture OpenStack
  ✅ Rôles des composants (Keystone, Nova, Neutron, Glance, Cinder)
  ✅ Flux de communication
  ✅ Premiers pas CLI

JOUR 2 : Le Lab
  ✅ DevStack (apprentissage rapide)
  ✅ Kolla-Ansible AIO (proche prod)
  ✅ Architecture réseau du lab
  ✅ Validation de l'installation

JOUR 3 : Administration
  ✅ Gestion des identités et quotas
  ✅ Infrastructure réseau SDN complète
  ✅ Cycle de vie des instances
  ✅ Volumes et snapshots

JOUR 4 : Avancé
  ✅ Orchestration avec Heat
  ✅ Méthodologie de dépannage
  ✅ Bonnes pratiques production
  ✅ Projet final 3-tiers
```

## 🎓 Certifications Recommandées Après ce Bootcamp

| Certification | Organisme | Niveau |
|---------------|-----------|--------|
| Certified OpenStack Administrator (COA) | OpenStack Foundation | Intermédiaire |
| Red Hat OpenStack Administration (CL210) | Red Hat | Avancé |
| Canonical OpenStack Operator | Canonical | Intermédiaire |

## 📖 Ressources pour Aller Plus Loin

- [Documentation Officielle OpenStack](https://docs.openstack.org/)
- [OpenStack Operations Guide](https://docs.openstack.org/operations-guide/)
- [Kolla-Ansible Docs](https://docs.openstack.org/kolla-ansible/latest/)
- [Heat Template Guide](https://docs.openstack.org/heat/latest/template_guide/)
- [OpenStack Security Guide](https://docs.openstack.org/security-guide/)
