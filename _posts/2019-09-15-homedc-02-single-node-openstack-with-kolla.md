---
layout: post
comments: true
title: Home DC 02&#58; Single-node OpenStack with Kolla
author: Igor D.C.
tags:
- Cloud
- OpenStack
- Kolla
- HomeDC
- Neutron
---

In the [the first Home DC post](/2018/12/homedc-01-planning-my-first-personal-dev-cloud), I introduced some of the hardware I made available for the purpose of deploying a home/lab cloud/cluster/datacenter (or many). I have some ideas about what to do with this hardware but will leave that to another day, another post. Today I want to give you an introduction to my simple recipe for deploying a single-node AIO (`all-in-one`) OpenStack based on kolla / kolla-ansible. I'll go through the hardware and network requirements and environment used. The [next post](/2019/09/homedc-03-installing-single-node-openstack-with-kolla) will walk through the steps to setup this deployment and outline some of the difficulties I faced until the deployment was easily reproducible.
<span style="display: block; text-align: center">![](/assets/datacenter02.jpg "Home DC 02"){:width="84%"}</span>

# Requirements

In order to proceed with this basic deployment, with the instructions I provide below, these are the essential things needed:
* 1 machine (physical or virtual)
  * running up-to-date Ubuntu 18.04
  * 16GB of RAM (at least) recommended and
  * 4 physical cores to account for at least a small-ish virtual machine (VM)
* 2 network interfaces
  * to connect to 2 different networks
  * **both** of the interfaces need to have access to the Internet (see Networking section below)
* Direct access to the Internet (proxy setup is **not** covered in this post)

# Networking

<span style="display: block; text-align: center">![](/assets/homedc02_network.png "Network diagram for this deployment"){:width="100%"}</span>
As mentioned in the Requirements above, **two network interfaces** are needed for this single-node AIO deployment based on kolla / kolla-ansible. In the figure above we can see two networks (green and orange clouds in the center) that connect to the two interfaces of the server.

The **Management Network**, in green, is going to be the primary network for accessing the OpenStack API and also to SSH into the server (before or after deploying). Additionally, this is the interface that will give Internet connectivity to the host itself, not its guests/instances/VMs, in order to receive security updates, etc.

The **External/Flat network**, to become a physical/provider Neutron network, in orange, is going to be how the VMs will reach the Internet. The VMs can either have an IP address directly on this network, a floating IP or neither. Nonetheless, Internet connectivity for the instances will have to go through this network.

On the right, we see the **server** that will be running OpenStack, nicknamed **z820** since this is a 2nd generation [HP Z820 workstation](https://www8.hp.com/h20195/v2/getpdf.aspx/c04111526.pdf?ver=36). This is *Node 5* outlined in the [Home DC 01](/2018/12/homedc-01-planning-my-first-personal-dev-cloud) post.

On the left, there's an **OpenWrt-powered switch/router** ([Linksys WRT3200AC](https://openwrt.org/toh/linksys/wrt_ac_series#wrt3200acm)) that I use to easily access and isolate the whole Home DC environment, including wireless and additional security mechanisms that I might cover in the future. Of course, there's some sort of Internet connection reachable from this OpenWrt device.

# In more detail

**Two VLANs** are configured in the OpenWrt switch, respectively referring to the `br-lan` and `br-lan3` network interfaces that I created and are managed by the operating system. The OpenWrt router has the IP addresses `192.168.2.1` and `192.168.3.1` respectively configured for the VLANs created, which match the network ranges outlined in the figure for the Management and the External/Flat networks.

The OpenWrt device additionally runs a **DHCP server** on the `br-lan` network (referred to as VLAN 1 or simply Management Network from now on). It **does not** run a DHCP server on the `br-lan3` network (referred to as VLAN 3 or simply External Network from now on).

IP addressing in the External Network will be handled by Neutron, the networking brains and brawn of OpenStack.

Server z820 will receive its management IP address by DHCP from the OpenWrt device. The actual interface for my specific machine is `enp1s0`. However, I have set up the OpenWrt DHCP server to always give the same IP address to z820 since we don't really want this address to ever change (high-availability, load balancing, etc. aside). The external interface, `ens5s3`, is marked with a `.Y` IP address because this interface may take different IP addresses, even simultaneously, since multiple VMs can be "attached" to it.

In the figure we can see that the z820 server is marked with "SR-IOV". Although the machine has a 4-port 1G SR-IOV card, in the context of the single-node AIO deployment, this card is not in use.

# Proxy considerations

In this first phase of the single-node AIO deployment, proxy configurations with the purpose of accessing the Internet, will not be covered. The user should evaluate whether this is necessary for their particular environment (typically not necessary) and make system configurations to account for the need of a proxy server. The user can also try my [`chameleonsocks-auto-proxy.sh`](https://github.com/igordcard/homedc/blob/master/scripts/redsocks/chameleonsocks-auto-proxy.sh) script which relies on chameleonsocks and redsocks to automatically forward all traffic to a SOCKS proxy.

# Next post

The next post, [Home DC 03](/2019/09/homedc-03-installing-single-node-openstack-with-kolla), covers the actual steps for installing OpenStack in the single machine.

My initial intention was to go through everything related to the single node AIO kolla OpenStack deployment in a single blog post: requirements, networking, steps, problems, etc. However, as I like to keep posts short whenever I can, I decided to have the actual setup steps be in their own post.

Let me know your thoughts below!

# Additional reading

[How To Get Started With OpenStack](https://www.openstack.org/software/start/)

[OpenStack Quick Start with kolla-ansible](https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html)

[OpenStack Networking](https://docs.openstack.org/mitaka/networking-guide/intro-os-networking.html)

[OpenStack provider network](https://docs.openstack.org/newton/install-guide-ubuntu/launch-instance-networks-provider.html)