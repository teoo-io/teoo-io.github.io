---
layout: post
title: "OpenStack Deployment - Overview"
date: 2021-01-30 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, charms, juju, maas]
---
The purpose of this guide is to aid in the replication and maintenance of our OpenStack cloud by documenting the deployment process.

# Versions
The software versions used in the BSU environment are as follows:
* Ubuntu 20.04 LTS (Focal) for the MAAS server, Juju client, Juju controller, and all cloud nodes (including containers)
* MAAS 2.9.2
* Juju 2.7.6
* OpenStack Wallaby

# Hardware Layout (5 Nodes)

| Name          | Description   |
| ------------- |---------------|
| Client Machine | Machine used to SSH into the MAAS Controller, or MAAS nodes. |
| MAAS Controller | This machine is not part of the OpenStack cloud - it simply manages the “metal” of the cloud by power-cycling machines and network booting and deploying OS images. |
| JuJu Controller | One of the nodes managed by MAAS. Not to be confused with the JuJu application that lives within the MAAS Controller. The JuJu Controller orchestrates the commands given to the JuJu application and hosts the JuJu dashboard. |
| node0-3 | These MAAS managed nodes (tagged “compute”) are where the OpenStack Charm containers get deployed to. These nodes form the actual OpenStack cloud. |


# Deployment Process

The first step in deployment is to configure the hardware (servers) so they can play nicely with the MAAS Application later in the deployment process. It is important to ensure all minimal hardware requirements are met.

Next, there needs to be some thought put into network architecture, and the Network Set-Up section lists some of the considerations.

With hardware and network configuration done, Ubuntu Server 20.04 LTS needs to be installed on a server separate from the OpenStack cluster - this server is referred to as the “MAAS Controller”. The MAAS Application is then installed on the MAAS Controller using the Snap Store, and is then configured appropriately.

The servers that are part of the OpenStack cluster (which are referred to as JuJu Controller, Node0, Node1, Node2, and Node3) need to then be enlisted on MAAS, and set to a “ready” state to be “deployed” later on by JuJu. This process is covered in the MAAS Enlisting section.

Before any JuJu magic can happen, the JuJu Application needs to be installed on the MAAS Controller using the Snap Store - all JuJu commands will be run on the MAAS Controller command line, but will be orchestrated by the JuJu Controller.

To actually deploy the JuJu Controller hardware (and nodes later on), the JuJu Application needs to have API access to MAAS. This quick set-up process is outlined in the JuJu Configuration section.  And once JuJu can control MAAS-managed hardware through API, the actual JuJu controller can be bootstrapped and deployed - this is done through command line on the MAAS Controller as described in the JuJu Deployment section.

With the JuJu Controller deployed, the OpenStack installation can begin. This (lengthy and tedious) process is described step-by-step under “Open Stack Installation”. The OpenStack cloud then needs to be configured for non-admin access and use, and managed for maintenance, scalability and power events - all of which is outlined in detail in the OpenStack Management section.



# [Next Section: Hardware](https://bsu-cybersecurity.github.io/posts/openstack-deployment-hardware/)
