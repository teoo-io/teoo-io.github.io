---
layout: post
title: "OpenStack Deployment - Network"
date: 2021-01-28 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, charms, juju, maas]
---
# Architecture
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

## Serving DHCP
We have the option to let our router or MAAS serve DHCP, and both options have their implications.

### Letting MAAS Serve DHCP
When MAAS is serving DHCP, MAAS will automatically discover a server that is PXE booting on the same network. When a server is booting over PXE, MAAS takes over by providing a boot image and automatically listing the node under “Machines” on the MAAS Application GUI - and if IPMI was set up correctly, MAAS will automatically configure the IPMI. Though this is very convenient, the downside however, is that the Firewall/Router won’t manage DHCP leases.

### Letting the Firewall/Router Serve DHCP
Alternatively, we can let the router serve DHCP. The main consideration here (as far as MAAS is concerned) is that we will have to manually enlist new servers by MAC and IP address, which isn’t a super tedious process but it is something to consider.



# [Next Section: MAAS](https://bsu-cybersecurity.github.io/posts/openstack-deployment-maas/)
