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
cd devstack
vi localrc

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
Install DevStack
```bash
./stack.sh
```

### Install Magnum

### Start Magnum

### Testing Magnum


## Limitations


## Contributors
