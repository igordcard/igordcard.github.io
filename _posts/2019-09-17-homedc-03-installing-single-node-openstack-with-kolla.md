---
layout: post
comments: true
title: Home DC 03&#58; Installing single-node OpenStack with Kolla
author: Igor D.C.
tags:
- Cloud
- OpenStack
- Kolla
- HomeDC
- Neutron
---

In the previous post, [Home DC 02](/2019/09/homedc-02-single-node-openstack-with-kolla), I've outlined the requirements and an environment/scenario for a single-node AIO kolla / kolla-ansible OpenStack deployment, the first piece in the Home DC. Now, I'm going to walk through the actual actions taken on the single server to get to the point where OpenStack is running.
<span style="display: block; text-align: center">![](/assets/datacenter03.jpg "Home DC 02"){:width="84%"}</span><br/>

# Recap
We require two networks to be connected to the machine, with both being able to access the Internet. IP addressing on the External/Flat network should be left to the OpenStack machine to handle, although the other side of the network (the default gateway) should have its own IP address previously set.
<span style="display: block; text-align: center">![](/assets/homedc02_network.png "Network diagram for this deployment"){:width="100%"}</span>

# Deployment

The following sections explain, step by step, how to complete this deployment.
The instructions are for **OpenStack Stein** on **Ubuntu 18.04**, with kolla / kolla-ansible.
If you don't need any more words, you can simply view/copy the full set of commands on my GitHub: [kolla-prep-stein-bionic-source-v3.sh](https://github.com/igordcard/homedc/blob/master/scripts/kolla/stein/bionic/kolla-prep-stein-bionic-source-v3.sh) and ignore the rest of this post.

# Preparations

Start by logging in as a normal user (non-root) and cd into the home folder.
From there, a few important variables should be set up.

`DCNAME` is the name of your deployment (will be used as the python virtualenv for ansible). 

`INT_IP` is the IP address of the server in the Management Network. Make sure it doesn't change later.

`INT_IF` is the interface name in the Management Network.

`EXT_IF` is the interface name in the External Network.

`EXT_NET`, `EXT_MASK` and `EXT_GW` define the expected [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) and default gateway of the External Network.

`EXT_RANGE_START` and `EXT_RANGE_END` define the pool of addresses that Neutron can allocate for the External Network.

At this point, if needed, proxy configurations may be done. You can check [chameleonsocks-auto-proxy.sh](https://github.com/igordcard/homedc/blob/master/scripts/redsocks/chameleonsocks-auto-proxy.sh) for transparently forwarding traffic to a SOCKS proxy.

``` bash
DCNAME='homedc'
INT_IP='192.168.2.20'
INT_IF='enp1s0'
EXT_IF='ens5s3'
EXT_NET='192.168.3.0'
EXT_GW='192.168.3.1'
EXT_MASK='24'
EXT_RANGE_START='192.168.3.50'
EXT_RANGE_END='192.168.3.99'
cd
```

# Dependencies

Install a few needed packages and (**warning**) remove any apt-installed pip. Also install docker-ce in order to run the OpenStack services in containers, with the kolla container images. Then get upstream pip and set up the virtualenv for ansible. Finally clone kolla and kolla-ansible (this is a kolla source installation, better for development and OK for the kind of home deployment I'm looking for here).

``` bash
# python stuff
sudo apt-get update
sudo apt-get install build-essential python-dev libffi-dev gcc libssl-dev python-selinux python-setuptools -y
sudo apt-get remove python-pip python3-pip python-virtualenv python3-virtualenv -y

# docker-ce
sudo apt-get remove docker docker-engine docker.io containerd runc -y
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io -y

wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
pip install --user virtualenv

virtualenv --clear $DCNAME
source $DCNAME/bin/activate
pip install -U pip ansible

# kolla-ansible source installation (better for development)
git clone https://github.com/openstack/kolla
git clone https://github.com/openstack/kolla-ansible
pip install -r kolla/requirements.txt
pip install -r kolla-ansible/requirements.txt
```

# Generic configurations

The following simply initializes a bunch of necessary configuration files, directories, etc. One important part is the last line, where `/etc/kolla/passwords.yml` is generated. Later you should access this file to copy the OpenStack admin password. This is also where the most important kolla configuration file is copied, `/etc/kolla/globals.yml`.

``` bash
# init configs
rm -rf /etc/kolla
sudo mkdir -p /etc/kolla
sudo cp -r kolla-ansible/etc/kolla/* /etc/kolla
sudo chown -R $USER:$USER /etc/kolla
cp kolla-ansible/ansible/inventory/* .

# ansible configuration
sudo mv /etc/ansible/ansible.cfg /etc/ansible/ansible.cfg.bak
sudo mkdir /etc/ansible
sudo touch /etc/ansible/ansible.cfg
sudo bash -c 'cat <<EOT > /etc/ansible/ansible.cfg
[defaults]
host_key_checking=False
pipelining=True
forks=100
EOT'

# generate new /etc/kolla/passwords.yml
./kolla-ansible/tools/generate_passwords.py
```

# Knobs

The following sed commands replace a few default/commented variables in `/etc/kolla/globals.yml` (the main file providing kolla knobs). You can change them to other values to your preference. Most of the values already come from the variables at the beginning of this guide.

This section has a higher-risk of failure due to the possible changes that can be made. Make sure all substitutions succeeded as soon as you begin troubleshooting any issue.

``` bash
sed -i "s/#kolla_base_distro: \"centos\"/kolla_base_distro: \"ubuntu\"/" /etc/kolla/globals.yml
sed -i "s/#kolla_install_type: \"binary\"/kolla_install_type: \"source\"/" /etc/kolla/globals.yml
sed -i "s/#openstack_release: \"\"/openstack_release: \"stein\"/" /etc/kolla/globals.yml
sed -i "s/#kolla_internal_vip_address: \"10.10.10.254\"/kolla_internal_vip_address: \"$INT_IP\"/" /etc/kolla/globals.yml
sed -i "s/#network_interface: \"eth0\"/network_interface: \"$INT_IF\"/" /etc/kolla/globals.yml
sed -i "s/#neutron_external_interface: \"eth1\"/neutron_external_interface: \"$EXT_IF\"/" /etc/kolla/globals.yml
sed -i "s/#enable_haproxy: \"yes\"/enable_haproxy: \"no\"/" /etc/kolla/globals.yml
```

# Ready

When all of the above has completed, you should tell kolla-ansible to run the `bootstrap-servers` and `prechecks` commands against the `all-in-one` inventory file (which got copied earlier) which will attempt to make sure that everything needed to actually deploy OpenStack on the particular machine is in place.

If you happen to run one command much later than the other, there's a chance that sudo has expired and you might see some strange errors as kolla-ansible runs and, eventually, fails. Just run sudo manually, e.g. `sudo ls`.

``` bash
./kolla-ansible/tools/kolla-ansible -i ./all-in-one bootstrap-servers
./kolla-ansible/tools/kolla-ansible -i ./all-in-one prechecks
```

# Deploy

If the commands above succeed, then it's time to deploy OpenStack with kolla! Go ahead and run the `deploy` command. this should take a few minutes.

``` bash
./kolla-ansible/tools/kolla-ansible -i ./all-in-one deploy
```

# Final steps

If kolla deploys successfully, `post-deploy` should be called next, which will run some additional steps and create the `admin-openrc.sh` file. This file should be sourced in the shell so that we can talk to the OpenStack API to create some initial resources, networks, subnets, virtual router, flavors, a cirros image, etc.

Make sure that you are still inside the virtualenv originally created. The variables defined earlier should still be defined in the session. Also, the OpenStack CLI tools will be installed in this virtualenv and not on the main python env of the machine.

``` bash
./kolla-ansible/tools/kolla-ansible -i ./all-in-one post-deploy
source /etc/kolla/admin-openrc.sh
# configure public external/flat network addressing
sed -i "s/10.0.2.0\/24/$EXT_NET\/$EXT_GW/" kolla-ansible/tools/init-runonce
sed -i "s/start=10.0.2.150,end=10.0.2.199/start=$EXT_RANGE_START,end=$EXT_RANGE_END/" kolla-ansible/tools/init-runonce
sed -i "s/10.0.2.1/$EXT_GW/" kolla-ansible/tools/init-runonce
# CLI
pip install -U python-openstackclient python-glanceclient python-neutronclient
# create basic OpenStack resources (uses CLI above)
./kolla-ansible/tools/init-runonce
```

# Uninstall

To uninstall OpenStack, run the kolla-ansible `destroy` command, optionally remove the configurations at `/etc/kolla` (backup is recommended), get out of the virtualenv and delete it.

``` bash
# source $DCNAME/bin/activate
./kolla-ansible/tools/kolla-ansible -i ./all-in-one destroy --yes-i-really-really-mean-it
sudo rm -rf /etc/kolla/
deactivate
rm -rf $DCNAME        
```

# Full script

<p>The whole script is available <a href="https://github.com/igordcard/homedc/blob/master/scripts/kolla/stein/bionic/kolla-prep-stein-bionic-source-v3.sh">on GitHub</a>, just keep in mind that it's not designed to be run all at once. For that I might end up switching to Ansible later.</p>

# A note on Ansible

Ansible uses idempotent operations, i.e., the same task ran a second time does not change the result of the first time. A task is completed when a particular state is attained, rather than by blindly executing specific actions. As such, if you encounter failures while running the kolla-ansible commands and you believe they might have been caused by some sort of temporary issue, re-running the same kolla-ansible may get it to succeed. There's no need to try and rewind the parts that were successful in order to retry everything, due to the idempotent nature of Ansible and kolla-ansible.

Likewise, uninstalling kolla-ansible with the `destroy` command should cleanly remove OpenStack from the system, which is further helped by the fact that the OpenStack services run in containers (and mostly act in isolation).

# Additional reading

[Operating Kolla](https://github.com/openstack/kolla-ansible/blob/master/doc/source/user/operating-kolla.rst)

[Ansible Glossary](https://docs.ansible.com/ansible/latest/reference_appendices/glossary.html)

[How To Get Started With OpenStack](https://www.openstack.org/software/start/)

[OpenStack Quick Start with kolla-ansible](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)

[OpenStack Networking](https://docs.openstack.org/mitaka/networking-guide/intro-os-networking.html)

[OpenStack provider network](https://docs.openstack.org/newton/install-guide-ubuntu/launch-instance-networks-provider.html)