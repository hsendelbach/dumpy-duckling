# dumpy-duckling

#### Table of Contents

1. [Overview](#overview)
2. [In a Nutshell](#Nutshell)
3. [Usage - How to Start](#usage)
4. [Limitations - OS compatibility, etc.](#limitations)
5. [Contributors](#contributors)

## Overview
WARNING - This is a work in progress
Experiment with OpenStack Magnum (Kubernetes) inside a vagrant VM riding
on the Mac (Virtualbox)

## Nutshell
This project should allow us to accomplish the following:

  1) Install Vagrant
  2) Spin up a Trusty VM
  3) Install DevStack
  4) Install Magnum (Server and Client)
  5) Configure Magnum (Heat Kubernetes)
  6) Start Magnum
  7) Testing Magnum

## Usage


### Assumptions

* VirtualBox installed
* Mac OSx

### Install Vagrant

```bash
    add details here.
```
### Spin up Trusty VM
Clone this repository
```bash
git clone https://github.com/jfarschman/dumpy-duckling
```
Add an Ubuntu Trusty image
```bash
vagrant box add ubuntu/trusty64
```
Start up your VM.  Note, it's going to mount the . (present directory) as
an NFS mount on your Mac so you will be asked to authentication for the 
keychain.
```bash
cd dumpy-duckling
vagrant up
```

#### Notes on Vagrant

* vagrant up - will start your vm
* vagrant halt - will halt it
* vagrant reload - will reload
* vagrant destroy - completely removes it so you can start over
* vagrant provision - runs the vm.provision lines from the Vagrantfile
* vagrant reload --provision - will reload and run vm.provision
* vagrant ssh - will let you ssh into the VM.

### Install DevStack
Setup the basic packages
```bash
vagrant ssh
sudo apt-get update
sudo apt-get install -y vim git libmysqlclient-dev openvswitch-switch
```
Add the Kubernetes client
```bash
curl -O -L https://github.com/GoogleCloudPlatform/kubernetes/releases/download/v0.16.0/kubernetes.tar.gz
tar zxvf kubernetes.tar.gz
cd kubernetes/platforms/linux/amd64/
sudo cp -rp ./* /usr/local/bin/
```
Setup Network settings
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo /sbin/iptables-save -c | sudo tee -a /etc/iptables.rule
cat << _EOF_ | sudo tee -a /etc/network/if-pre-up.d/iptables_start
#!/bin/sh
/sbin/iptables-restore < /etc/iptables.rules
exit 0
_EOF_
sudo chmod +x /etc/network/if-pre-up.d/iptables_start
```
Get Devstack
```bash
cd /vagrant
git clone https://git.openstack.org/openstack-dev/devstack
```
Setup the localrc
```bash
cat << _EOF_ | sudo tee -a /vagrant/devstack/localrc
FLOATING_RANGE=192.168.19.0/24
Q_FLOATING_ALLOCATION_POOL="start=192.168.19.80,end=192.168.19.100"
PUBLIC_NETWORK_GATEWAY=192.168.19.1

Q_USE_SECGROUP=True
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=1000:1999
PHYSICAL_NETWORK=default
OVS_PHYSICAL_BRIDGE=br-ex

NETWORK_GATEWAY=10.11.12.1
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256

ADMIN_PASSWORD=openstack
MYSQL_PASSWORD=stackdb
RABBIT_PASSWORD=stackqueue
SERVICE_PASSWORD=$ADMIN_PASSWORD
SERVICE_TOKEN=tokentoken

disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service neutron

enable_service h-eng
enable_service h-api
enable_service h-api-cfn
enable_service h-api-cw

LOGFILE=$DEST/logs/devstack.log
DEST=/opt/stack
SCREEN_LOGDIR=$DEST/logs/screen
_EOF_
```
Add local.sh to run at the end of stack.sh
```bash
cat << _EOF_ | sudo tee -a /vagrant/devstack/localrc
sudo ifconfig br-ex 0.0.0.0
_EOF_
```
Install DevStack
```bash
cd /vagrant/devstack
./stack.sh
```

### Install Magnum
Magnum Server
```bash
cd /vagrant
git clone https://github.com/stackforge/magnum.git
cd magnum
tox -evenv -- echo 'done'
```
Magnum Client
```bash
cd /vagrant
git clone https://github.com/stackforge/python-magnumclient.git
cd python-magnumclient
tox -evenv -- echo 'done'
```
Configuration
```bash
sudo mkdir -p /etc/magnum/templates
cd /etc/magnum/templates
sudo git clone https://github.com/larsks/heat-kubernetes.git
cd /etc/magnum

cat << _EOF_ | sudo tee -a /etc/magnum/magnum.conf
[DEFAULT]
debug = True
verbose = True

rabbit_userid=stackrabbit
rabbit_password = stackqueue
rabbit_hosts = 127.0.0.1
rpc_backend = rabbit

[database]
connection = mysql://root:stackdb@localhost/magnum

[keystone_authtoken]
admin_password = openstack
admin_user = nova
admin_tenant_name = service
identity_uri = http://127.0.0.1:35357

auth_uri=http://127.0.0.1:5000/v2.0
auth_protocol = http
auth_port = 35357
auth_host = 127.0.0.1
_EOF_
```
Register image in Glance
```bash
cd /vagrant
curl -O https://fedorapeople.org/groups/heat/kolla/fedora-21-atomic-2.qcow2
source /vagrant/devstack/openrc admin admin
glance image-create \
  --disk-format qcow2 \
  --container-format bare \
  --is-public True \
  --name fedora-21-atomic \
  --file /vagrant/fedora-21-atomic-2.qcow2
```

Add keypair for Demo user
```bash
ssh-keygen
source /vagrant/devstack/openrc demo demo
nova keypair-add --pub-key ~/.ssh/id_rsa.pub default
```

Database work
```bash
mysql -h 127.0.0.1 -u root -pstackdb mysql <<EOF
CREATE DATABASE IF NOT EXISTS magnum DEFAULT CHARACTER SET utf8;
GRANT ALL PRIVILEGES ON magnum.* TO
    'root'@'%' IDENTIFIED BY 'stackdb'
EOF

cd /vagrant/magnum
source .tox/venv/bin/activate
pip install mysql-python
magnum-db-manage upgrade
```

### Start Magnum

magnum-api
```bash
cd /vagrant/magnum
source .tox/venv/bin/activate
magnum-api
```
magnum-conductor
```bash
cd /vagrant/magnum
source .tox/venv/bin/activate
magnum-conductor
```
python-magnumclient
```bash
cd /vagrant/python-magnumclient
source .tox/venv/bin/activate
magnum bay-list
```

### Testing Magnum
Attemt to create a Bay
```bash
NIC_ID=$(neutron net-show public | awk '/ id /{print $4}')
magnum baymodel-create --name default --keypair-id default \
  --external-network-id $NIC_ID \
  --image-id fedora-21-atomic \
  --flavor-id m1.small --docker-volume-size 5

magnum bay-create --name kbay --baymodel-id default
```
Create a Pod
```bash
magnum pod-create --bay-id 99cab72f-16a7-4564-8d73-d4497f51f557 \
  --pod-file redis-master.json
```

## Limitations


## Contributors
