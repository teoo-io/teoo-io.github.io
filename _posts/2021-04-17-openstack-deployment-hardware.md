---
layout: post
title: "Openstack Deployment - Hardware"
date: 2021-07-26 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, juju, maas]
---
# Configuration
## Dell PowerEdge Servers
BSU's SOC environment consists of Dell PowerEdge servers (no iDRAC license), so set-up steps are mostly identical across servers.

* Connect a display and keyboard (and mouse if preferred) to the server.
* Power on the server and wait until the boot menu on the top of the screen is displayed - at this point press F10 (Life Cycle Manager) on the keyboard and wait for the screen to load.
* Once in Life Cycle Manager we need to set up the following:

### RAID Controller
For setting up the raid controller, network storage and local storage have different virtual disk layouts.
- Local storage
  - Local storage will need virtual disks created. The first virtual disk will be comprised of two drives, and will be the 
![Desktop View](https://github.com/BSU-Cybersecurity/BSU-Cybersecurity.github.io/blob/main/images/lifecycle%20controller.jpg?raw=true)


Here, select “Network”.
![Desktop View](https://github.com/BSU-Cybersecurity/BSU-Cybersecurity.github.io/blob/main/images/idrac%20network.jpg?raw=true)


Since we do not have an iDRAC license we will use NIC1 for iDRAC instead of the dedicated iDRAC NIC. In our case we did not specify a “Fallover Network”.
![Desktop View](https://github.com/BSU-Cybersecurity/BSU-Cybersecurity.github.io/blob/main/images/LOM%201.jpg?raw=true)


Scrolling down, there are settings for registering iDRAC on DNS, which we disabled. We enabled DHCP for IPv4 as DHCP makes enlisting nodes on MAAS much easier, and we also enabled using DHCP to obtain DNS addresses.
![Desktop View](https://github.com/BSU-Cybersecurity/BSU-Cybersecurity.github.io/blob/main/images/iDrac%20on%20DNS%20disable.jpg?raw=true)


Lastly, and the whole point of setting up the iDRAC in the first place - we need to enable IPMI so that MAAS can power-cycle the machines. Below are the default settings - you can choose to assign a different privilege or encryption key but keep in mind that if you change the default, to make IPMI work you will need to specify both of these parameters in MAAS when setting up a new node.
![Desktop View](https://github.com/BSU-Cybersecurity/BSU-Cybersecurity.github.io/blob/main/images/ipmi%20lan.jpg?raw=true)


Now press the ESC key to return to the Life Cycle Manager main page and you will be asked to reboot. At this point you are done setting up iDRAC + IPMI settings.


### 3x RAID Virtual Disks
The first virtual drive is set up using the Wizard on the “Home” tab of Life Cycle Manager. This first step is ideal if the drives have foreign raid configurations already (if the drives came from another server). The wizard will walk you through the RAID configuration. Note: This is just one of three virtual disks we will set up so allocate physical drives accordingly.


The subsequent two virtual disks will be set up using the device manager itself. To access this, go to the “System Setup” tab of Life Cycle Manager and select “Advanced Hardware Configuration”. From there select Device Settings and then select the RAID controller from the list of devices.
  ![Desktop View](https://github.com/BSU-Cybersecurity/BSU-Cybersecurity.github.io/blob/main/images/Device%20Settings.jpg?raw=true)


Now press the ESC key to return to the Life Cycle Manager main page and you will be asked to reboot. At this point you are done setting up your three RAID virtual drives.

## PXE Boot
When MAAS wakes a server using IPMI, the server needs to perform a PXE Network boot by default. To enable this, go to the “System Setup” tab of Life Cycle Manager and select “Advanced Hardware Configuration”.
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


From here, select “System BIOS”, and once in System BIOS ensure the Boot Mode is set to UEFI. During our testing phase we also disabled Boot Sequence retry for debugging purposes. Next, click on “UEFI Boot Settings”.
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Once in UEFI boot settings select “UEFI Boot Sequence '', and use the +/- keys to move the NIC that you will be PXE booting from to the top of the list. In our case, we PXE boot from the same NIC that we perform IPMI over.
  ![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Now press the ESC key to return to the Life Cycle Manager main page and you will be asked to reboot. At this point you are done setting up PXE boot settings.
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Now press the ESC key to return to the Life Cycle Manager main page and you will be asked to reboot. At this point you are done setting up PXE boot settings.


# [Next Section: Network](https://bsu-cybersecurity.github.io/posts/openstack-deployment-network/)
