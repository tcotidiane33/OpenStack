## Deploy Production Ready OpenStack Using Kolla Ansible

In today’s tech landscape, cloud computing, especially with OpenStack, has become the backbone of modern infrastructure. OpenStack offers a powerful solution for organizations to create and manage their cloud environments, providing a range of services essential for scalable and flexible operations. However, deploying OpenStack for production requires careful consideration of scalability, reliability, and security.

Zoom image will be displayed
<img width="720" height="405" alt="image" src="https://github.com/user-attachments/assets/ebe4f03d-ca4f-4283-b4db-440cbf27e37e" />

Kolla Ansible is highly opinionated out of the box, but allows for complete customization. This permits operators with minimal experience to deploy OpenStack quickly and as experience grows modify the OpenStack configuration to suit the operator’s exact requirements. Kolla ansible provide production-ready containers and deployment tools for operating OpenStack clouds. This journal focuses on the practical aspects of deploying a production-ready OpenStack environment using Kolla Ansible. Kolla Ansible simplifies the deployment process through automation, making it easier for administrators to create a robust OpenStack cloud. Let’s dive into the world of cloud computing, where simplicity meets scalability.

More about kolla ansible: https://docs.openstack.org/kolla/latest/

OpenStack Topology
Zoom image will be displayed
<img width="720" height="281" alt="image" src="https://github.com/user-attachments/assets/c090c944-7aa4-4ecc-af0e-530e3862cd34" />

Ceph Topology
Zoom image will be displayed
<img width="720" height="205" alt="image" src="https://github.com/user-attachments/assets/bbe43968-fbba-4513-9b54-7b2281dd4fd3" />


IP Allocation
Zoom image will be displayed
<img width="1100" height="365" alt="image" src="https://github.com/user-attachments/assets/4cc8df16-65a5-4542-b74d-338bf2eb2ff5" />


Prerequisites:
5 Servers (3 controller+storage, 2 compute+storage)
Linux servers running Ubuntu 22.04
6 Subnets (internal net, self-service net, ceph public net, ceph cluster net, public net, provider net)
Internet Connection
Certain ports are open on the server
Ceph version Reef
OpenStack version Bobcat
There are 2 activities:
- First, Install ceph cluster with cephadm as external storage
Kolla Ansible does not provide support for provisioning and configuring a Ceph cluster directly. Instead, administrators should use a tool dedicated to this purpose like cephadm.cephadm is a utility that is used to manage a Ceph cluster.

- Second, Install OpenStack cluster integrated with ceph storage
After Ceph cluster already deployed as external storage, then OpenStack can be install with core services.

* Installation Ceph Cluster with Cephadm Utility
Let’s take a look at the steps required to set up Ceph Cluster using kubeadm. The steps look something like the following:

1. On deployer / Controller1 login as root and configure host on /etc/hosts
tee -a /etc/hosts<<EOF
### LIST SERVERS FOR CEPH CLUSTER 
172.16.3.21 controller1
172.16.3.22 controller2
172.16.3.23 controller3
172.16.3.24 compute1
172.16.3.25 compute2

172.16.4.21 controller1
172.16.4.22 controller2
172.16.4.23 controller3
172.16.4.24 compute1
172.16.4.25 compute2
EOF
2. Configure passwordless to root user, hostname and timezone from Deployer
ssh-keygen -t rsa
ssh-copy-id root@controller1
ssh-copy-id root@controller2
ssh-copy-id root@controller3
ssh-copy-id root@compute1
ssh-copy-id root@compute2

## Or copy manual file id_rsa.pub from Deployer node to 
## other nodes located in .ssh/authorized_keys
Set hostname and timezone for all nodes

for node in controller{1..3} compute{1..2}
do
  echo "=== Execute on $node ==="
  ssh root@$node hostnamectl set-hostname $node
  ssh root@$node timedatectl set-timezone Asia/Jakarta
  echo ""
  sleep 2
done
3. Update dan Upgrade package on Each Node
apt-get update -y; apt-get upgrade -y
4. Install Docker CE on Each Node
apt-get install apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor > /etc/apt/trusted.gpg.d/docker-ce.gpg
echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -sc) stable" > /etc/apt/sources.list.d/docker-ce.list
apt-get update; apt-get install docker-ce docker-ce-cli containerd.io -y
systemctl enable --now docker
5. Install cephadm Utility on Deployer / Controller1
Start from here execute only on Deployer / Controller1 as root

wget -q -O- 'https://download.ceph.com/keys/release.asc' | gpg --dearmor -o /etc/apt/trusted.gpg.d/cephadm.gpg
echo deb https://download.ceph.com/debian-reef/ $(lsb_release -sc) main > /etc/apt/sources.list.d/cephadm.list
apt-get update
apt-cache policy cephadm; apt-get install cephadm -y
6. Initialize Ceph Cluster Monitor
cephadm bootstrap --mon-ip=172.16.3.21 \
  --cluster-network 172.16.4.0/24 \
  --initial-dashboard-password=INPUTYOURPASSWORD \
  --dashboard-password-noupdate \
  --allow-fqdn-hostname | tee cephadm-bootstrap.log
The output should be like this:

Zoom image will be displayed
<img width="720" height="379" alt="image" src="https://github.com/user-attachments/assets/1d387d07-5302-426d-80ee-04a9b0d1a653" />

Enable Ceph CLI on Deployer and verify that ceph command is accessible
```bash 
/usr/sbin/cephadm shell \
  --fsid XXXXXXXXXXXXXXXXXXXXXXX \
  -c /etc/ceph/ceph.conf \
  -k /etc/ceph/ceph.client.admin.keyring
cephadm add-repo --release reef; cephadm install ceph-common
ceph versions
```
<img width="645" height="228" alt="image" src="https://github.com/user-attachments/assets/b1448047-8bdc-4410-acb0-4cd45aa6c6e7" />

7. Adding hosts to the cluster
Install the cluster’s public SSH key in the new host’s root user’s

for node in controller{1..3} compute{1..2}
do
  echo "=== Copying ceph.pub to $node ==="
  ssh-copy-id -f -i /etc/ceph/ceph.pub root@$node
  echo ""
  sleep 2
done
Tell Ceph that the new node is part of the cluster

for node in controller{1..3} compute{1..2}
do
  ceph orch host add $node
done
ceph orch host ls
8. Deploy OSDs to the cluster
ceph orch device ls
ceph orch apply osd --all-available-devices --method raw
9. Set label on all nodes
Set label mon on node controller1, controller2, andcontroller3

for node in controller{1..3} compute{1..2}
do
  ceph orch host label add $node mon
done
Set label osd on all nodes

for node in controller{1..3} compute{1..2}
do
  ceph orch host label add $node osd
done
ceph orch host ls
10. Create pool for OpenStack
for pool_name in volumes images backups vms
do
  ceph osd pool create $pool_name
  rbd pool init $pool_name
done
11. Create ceph keyring
ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images' -o /etc/ceph/ceph.client.glance.keyring
ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=images' -o /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=images' -o /etc/ceph/ceph.client.nova.keyring
ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups' -o /etc/ceph/ceph.client.cinder-backup.keyring
Now verify ceph cluster and see everything is great and check Ceph dashboard, access IP address of Controller1 https://controller1:8443/

Zoom image will be displayed
<img width="720" height="383" alt="image" src="https://github.com/user-attachments/assets/86534b82-8f7a-4f62-9f31-c7d89e80dda5" />

ceph status; ceph osd tree; ceph df
ceph orch ps; ceph osd pool ls
ls -lh /etc/ceph/
Zoom image will be displayed
<img width="720" height="299" alt="image" src="https://github.com/user-attachments/assets/4792659a-0eca-4e99-90a1-8d999a219f0e" />

* Install OpenStack with Kolla ansible
Let’s take a look at the steps required to install OpenStack using deployment tools kolla-ansible. The steps look something like the following:

1. On deployer/Controller1, configure host for OpenStack
Execute only on Deployer / Controller1
```bash 
tee -a /etc/hosts <<EOF
### LIST SERVERS FOR OPENSTACK
172.16.1.21 controller1 controller1.internal.achikam.net
172.16.1.22 controller2 controller2.internal.achikam.net
172.16.1.23 controller3 controller3.internal.achikam.net
172.16.1.24 compute1 compute1.internal.achikam.net
172.16.1.25 compute2 compute2.internal.achikam.net
172.16.1.55 internal.achikam.net

10.8.60.21 controller1.public.achikam.net
10.8.60.22 controller2.public.achikam.net
10.8.60.23 controller3.public.achikam.net
10.8.60.24 compute1.public.achikam.net
10.8.60.25 compute2.public.achikam.net
10.8.60.55 public.achikam.net
EOF
```
2. Update System, Install Python build and virtual environment dependencies on Deployer / Controller1
apt-get update -y
apt-get install python3-dev libffi-dev gcc libssl-dev python3-selinux python3-setuptools python3-venv -y
Create a virtual environment and activate it

python3 -m venv kolla-venv
echo "source ~/kolla-venv/bin/activate" >> ~/.bashrc
source ~/kolla-venv/bin/activate
3. Install ansible, kolla-ansible and its dependencies using pip
Refer to this official documentation for proper version match with kolla ansible requirements

pip install -U pip 
pip install 'ansible-core>=2.14,<2.16'
ansible --version
pip install git+https://opendev.org/openstack/kolla-ansible@master
4. Create ansible configuration
Create the /etc/kolla directory and update the ownership

mkdir -p /etc/kolla
chown $USER:$USER /etc/kolla
Copy fileglobals.yml and passwords.yml to /etc/kolla directory

cp -r ~/kolla-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp ~/kolla-venv/share/kolla-ansible/ansible/inventory/* .
Install Ansible Galaxy dependencies

kolla-ansible install-deps
Create file ansible.cfg

mkdir /etc/ansible
tee /etc/ansible/ansible.cfg<<EOF
[defaults]
host_key_checking=False
pipelining=True
forks=100
EOF
5. Modify inventory multinode configuration
Backup original file multinode

cp multinode multinode.bak
Edit file multinode, it will looks like below

```bash 
cat multinode
----
[control]
controller1
controller2
controller3

[network]
controller1
controller2
controller3

[compute]
compute1
compute2

[monitoring]
controller1
controller2
controller3

[storage]
controller1
controller2
controller3
compute1
compute2

[deployment]
localhost       ansible_connection=local
...
```
Test connection using ansible and ping module

ansible -i multinode all -m ping
6. Generate Password for every openstack services and TLS Certificate
kolla-genpwd
kolla-ansible -i multinode certificates
7. Create Directory for Kolla Ansible Configuration
Create kolla config directory for nova, glance, and cinder
```bash 
mkdir /etc/kolla/config
mkdir /etc/kolla/config/nova
mkdir /etc/kolla/config/glance
mkdir -p /etc/kolla/config/cinder/cinder-volume
mkdir /etc/kolla/config/cinder/cinder-backup
```
Copy file ceph.conf and ceph keyring to kolla config directory

```bash 
cp /etc/ceph/ceph.conf /etc/kolla/config/cinder/
cp /etc/ceph/ceph.conf /etc/kolla/config/nova/
cp /etc/ceph/ceph.conf /etc/kolla/config/glance/
cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/
cp /etc/ceph/ceph.client.nova.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-backup/
cp /etc/ceph/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup/
```
for node in controller{2..3} compute{1..2}
do
  scp -r /etc/ceph/ root@$node:/etc/
done
8. Create Main Configuration File for Kolla Ansible using globals.yml
Edit globals.yml file then verify main configuration with command
grep -v "#" /etc/kolla/globals.yml | tr -s [:space:]
```bash 
### /etc/kolla/globals.yml
---
kolla_base_distro: "ubuntu"
openstack_release: "master"
kolla_internal_vip_address: "172.16.1.55"
kolla_internal_fqdn: "internal.achikam.net"
kolla_external_vip_address: "10.8.60.55"
kolla_external_fqdn: "public.achikam.net"
kolla_external_vip_interface: "ens7"
api_interface: "ens3"
tunnel_interface: "ens4"
neutron_external_interface: "ens8"
neutron_plugin_agent: "ovn"
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_copy_ca_into_containers: "yes"
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
kolla_enable_tls_backend: "yes"
enable_openstack_core: "yes"
enable_cinder: "yes"
enable_fluentd: "no"
enable_neutron_provider_networks: "yes"
ceph_glance_user: "glance"
ceph_glance_keyring: "client.glance.keyring"
ceph_glance_pool_name: "images"
ceph_cinder_user: "cinder"
ceph_cinder_keyring: "client.cinder.keyring"
ceph_cinder_pool_name: "volumes"
ceph_cinder_backup_user: "cinder-backup"
ceph_cinder_backup_keyring: "client.cinder-backup.keyring"
ceph_cinder_backup_pool_name: "backups"
ceph_nova_keyring: "client.nova.keyring"
ceph_nova_user: "nova"
ceph_nova_pool_name: "vms"
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
nova_compute_virt_type: "kvm"
neutron_ovn_distributed_fip: "yes"
...
```
9. Deploy OpenStack
After configuration is set, we can proceed to the deployment phase. First we need to setup basic host-level dependencies, like docker. Kolla Ansible provides a playbook that will install all required services in the correct versions.

```bash 
### Bootstrap servers with kolla deploy dependencies
kolla-ansible -i multinode bootstrap-servers

### Do pre-deployment checks for hosts
kolla-ansible -i multinode prechecks

### Finally proceed to actual OpenStack deployment
kolla-ansible -i multinode deploy

### Do post-deploy after OpenStack was successfuly deployed
kolla-ansible -i multinode post-deploy
```
10. Configure OpenStack Client
Add root ca to ca-certificates
```bash 
cat /etc/kolla/certificates/ca/root.crt | sudo tee -a /etc/ssl/certs/ca-certificates.crt
```
Add CA Path to file RC admin-openrc.sh
```bash 
echo "export OS_CACERT=/etc/ssl/certs/ca-certificates.crt" >> /etc/kolla/admin-openrc.sh
```
Install OpenStack Client
```bash 
pip3 install python-openstackclient
```
Verify OpenStack client
```bash 
openstack --version
```
11. Verify OpenStack cluster, see everything is great and check Ceph Cluster was integrated with OpenStack
```bash 
source ~/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh
openstack endpoint list
openstack service list
openstack compute service list; openstack network agent list
openstack volume service list; cinder get-pools
```
Zoom image will be displayed
<img width="720" height="355" alt="image" src="https://github.com/user-attachments/assets/fe31bc3d-0770-42fa-83b3-2bba0ae056ee" />


Access horizon using kolla_external_fqdn which is in this case is https://public.achikam.net

Zoom image will be displayed
<img width="720" height="373" alt="image" src="https://github.com/user-attachments/assets/3c055a8e-dfee-4fe0-a868-c5af48995abe" />


Zoom image will be displayed
<img width="720" height="388" alt="image" src="https://github.com/user-attachments/assets/b74a48c4-0e75-48ae-80ac-fe4b9370b0a0" />

Zoom image will be displayed
<img width="1100" height="598" alt="image" src="https://github.com/user-attachments/assets/f435fd5f-5ac7-485a-9c12-6237e23e089c" />

[OPTIONAL] there is an after post-deploy kolla ansible here Using OpenStack After Kolla Ansible Deployment

References :

https://docs.openstack.org/kolla-ansible/latest/
https://kifarunix.com/install-and-setup-ceph-storage-cluster-on-ubuntu-2204
#OpenStack #kolla #ansible #ceph #CloudComputing #PrivateCloud #OpenSource
