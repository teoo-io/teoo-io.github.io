---
layout: post
title: "OpenStack Deployment - MAAS"
date: 2021-01-27 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, charms, juju, maas]
---
# Controller OS
One server that isn’t part of the OpenStack cluster (but is on the same network) needs to be the designated “MAAS Controller”. The MAAS Controller will “manage” the hardware (servers), meaning it can power on/off + deploy operating system images to each of the servers networked in the OpenStack cluster.
## Installing the Operating System
Though most Debian Linux distributions would work, our MAAS Controller runs Ubuntu Server Focal 20.04 LTS.
## A Few Pointers
* For troubleshooting purposes, it is advisable to manually assign an IP during the installation of
  Ubuntu rather than automatically using DHCP. The reason why, is if the DHCP service within MAAS stops working, the MAAS controller won't have an IP, and you won't be able to SSH into it - and that brings up the next point below.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }



* It is infinitely easier to manage the MAAS Controller over SSH, however, Ubuntu Server comes out of the box with UFW, so SSH needs to be allowed through the firewall. To allow ssh, input the following command.
```bash
sudo ufw allow ssh
```

# Installation
With Ubuntu configured on the MAAS Controller, the MAAS application needs to be installed using the Snap Store by inputting this command:

```bash
sudo snap install --channel=2.9/stable maas
```
MAAS needs a database backend, which can be installed using the following commands.

```bash
sudo apt update -y
sudo apt install -y postgresql
```

A user for the database then needs to be created. Input the following command - the $VARIABLE’s need to be replaced appropriately and documented for future reference.
```bash
sudo -u postgres psql -c "CREATE USER \"$MAAS_DBUSER\" WITH ENCRYPTED PASSWORD '$MAAS_DBPASS'"
```
Then the database needs to be created with the user created above. Run the following command replacing the $VARIABLE’s.

```bash
sudo -u postgres createdb -O "$MAAS_DBUSER" "$MAAS_DBNAME"
```
A postgres config file now needs to be edited to reflect the new database user and database name. At the time of writing this guide, the latest stable version of postgres is version 12, so the following command will use vim to open the file in this directory  - `````/etc/postgresql/12/main/pg_hba.conf````` using root privilege. In the event this doesn’t work, manually navigate to this directory, you likely have a different version of postgres installed so the ‘12’ in the directory may be a different number.


```bash
sudo vim /etc/postgresql/12/main/pg_hba.conf
```

With the ```pg_hba.conf``` file open, add the following line at the end, and save the file.


```bash
host    $MAAS_DBNAME    $MAAS_DBUSER    0/0     md5
```

Now it is time to initialize the MAAS application using the command below. Replace the variables as appropriate, and since we are running the MAAS application on our MAAS  Controller, the `````$HOSTNAME````` should be set to ```localhost```.


```bash
sudo maas init region+rack --database-uri "postgres://$MAAS_DBUSER:$MAAS_DBPASS@$HOSTNAME/$MAAS_DBNAME"
```

Note that the IP of the web GUI is printed out on the terminal during this step. Press enter unless you want to make any changes. If you specified that the ```$HOSTNAME``` was ```localhost```, the IP of the web GUI is the same as that of the MAAS Controller machine.

Finally, a MAAS admin user needs to be created.

```bash
sudo maas createadmin
```

You will be prompted to input user credentials - these will be used to log into the web GUI so make sure to document these for future use.


Now check the status of the MAAS application using the following command


```bash
sudo maas status
```

The output should be similar to the following.

```bash
sudo maas status
bind9                        RUNNING   pid 22236, uptime 0:01:02
dhcpd                        STOPPED   Not started
dhcpd6                       STOPPED   Not started
http                         RUNNING   pid 22461, uptime 0:00:42
ntp                          RUNNING   pid 22372, uptime 0:00:45
proxy                        RUNNING   pid 22604, uptime 0:00:34
rackd                        RUNNING   pid 22241, uptime 0:01:02
regiond                      RUNNING   pid 22248, uptime 0:01:02
syslog                       RUNNING   pid 22366, uptime 0:00:45

```
If any  services are not running, follow this process to re-initialize MAAS.

At this point the MAAS web GUI should be accessible from a web browser on a machine sharing the same network as the MAAS Controller.

# Initial Set-Up

Access the MAAS web GUI from your local browser. You should be able to access it as long as you are on the same network as the MAAS Controller and MAAS is running.
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Log in to the MAAS web GUI and follow the set-up process. The steps are mostly straight-forward.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


## Import SSH Keys
In the next step of the initial set-up, you will be asked to input an ssh-key so that you can ssh into the machines MAAS deploys using RSA key authentication instead of a password. Grab the machine you intend to use to SSH into the MAAS-deployed nodes and retrieve the RSA key pair of that machine. If you are on a Linux machine and you have already generated SSH keys, the key can usually be retrieved from the default directory by entering the following command.

```bash
sudo cat /home/$USERNAME/.ssh/id_rsa.pub
```

More than likely though, you will need to generate your “private/public key pair”. To generate the RSA key pair enter the following command. Follow the shell prompts and take note of the directory and passphrase (if any) used.

```bash
ssh-keygen -t rsa -b 4096
```
If you saved the RSA key pair to the default directory you will be able to retrieve your public key with the following command.

```bash
sudo cat /home/$USERNAME/.ssh/id_rsa.pub
```
The output of this will look something like this:

```bash
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC/2sats4R9ino4qZfNiPFL+PYAVjY6vdRP9G9KJZPyxeOhbcpUPQ88gf2TpjEhy9CICweJ9Wt71uhkyh7iL4nW2ZN1oPyF9jzJTTfxPyAGjV6cOOPC1ispOgmFu3ZklGpRXAaNYsgIpp5FGGyioIAatzDuBjHffYEEXqmj6qkamRhCZZdMl7EBxa0rwsTlOfdY72ohpqwqUQr6QFMpvnSO1VFTjQJIXSvpDBigdhYsL0TxH+4S9UG+ywBQunhaOpX34Z6bIQT1rfZbDEfMh/C6qcQQSFghclhCjvqn2V7FNkiYcIuImf18xLQMnZU2NKRM9x2inhdjWUXvpihJbVzZuZ03AAm0NF1H2va3fBMJrTAXKLf4wTj1k+841+d59EVD6eUGW94p3sAzkC39FQMizv1PovFCSFFjNt5zFJWEFjMNYJWvLmx2DUnEnn7Qsmjw8U66YcI03FxWwW1sab+hbsS9y5rjOlUIDc1zjHlfRZmqf0obAghcQ852WvRYKv6TJ/WE2RQtUXg0/lJjtG70+mnrOxqAE3Pk2kT2gZumhk01VlvZCVBz8bLNdZ3r5jjEVYJ/MChQW7Uv828EtA1fayXqXLtIrYZxCPcvA99GKN3/xT0HNGWSYdrelVf0CZ207aVjJuP2mfDr5DWZ9PhXxiiz/hF6TP1abKOqNfsb8w== controller@maas
```

You will need to copy this output and paste it into the MAAS web GUI.
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


With the RSA public key imported, continue through the initial set-up. Once you land on the page below there are a couple things that still need to be configured.
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


# Configuration
## Enable DHCP
This step only applies if you’ve decided to let MAAS serve DHCP, as described in the Serving DHCP section of this guide.

To enable DHCP click on the “Controllers” tab at the top, and select the controller (called maas.maas in this case).
![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


In here click on “VLANs” and click on “VLAN” “untagged”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Scroll down to the DHCP section and click on “Enable DHCP”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Now, click on “Configure DHCP”

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

At this point DHCP is configured.


## Configure DNS
Early in our deployment we encountered some issues where MAAS would commission the servers but the servers would not be able to reach the internet to install APT packages. We figured out that, though we had specified the DNS during the initial set-up process, we also needed to specify the DNS within the controller VLAN itself. To set this up go to the “Controllers” tab in the MAAS web GUI, and select your MAAS controller (“maas.maas” in this case).

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


After selecting the controller from the list, click on “VLANs” and then click on the subnet listed under “subnets” (in this case it is “192.168.1.0/24”).

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Now, find “Subnet summary” and click on “Edit”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


Specify the appropriate DNS in the DNS text field and click “Save summary”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

## Disable MAAS Proxy
MAAS uses a proxy to download boot images and provision packages for APT and YUM. This has caused issues in our environment where MAAS does not see the hardware inventory of our nodes and, though it manages to “enlist” the nodes, it fails to deploy them. To disable the proxy from the web GUI, go to the “Settings” tab at the top and click the “Proxy” section on the left hand side. Then select the “Don’t use a proxy” option.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

With DHCP handled, proxy disabled, and DNS properly configured, you are now ready to enlist nodes.

## Enlisting Nodes (MAAS serving DHCP)
Enlisting a node when MAAS is serving DHCP is very simple. All you have to do to is power the server on - as long as it is on the same network as the MAAS Controller, and IPMI + PXE boot are properly configured (as specified in Hardware Configuration), the node will boot under MAAS instruction and will be listed in the web GUI under “Machines” as shown below. Once the node is enlisted and in “New” state, the physical machine will power down.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

It can be challenging to identify what physical node corresponds to each machine listed in the MAAS web GUI (especially if the nodes have similar hardware configuration), so it is best to enlist one node at a time.

When enlisting each node, wait until MAAS says the node is in a “New” state. At that point click on the node and on the top left corner rename it to make it more easily identifiable in the future. In this case we are naming this node “juju” since it will be the JuJu Controller node.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

Next, click on the “Configuration” tab, add the appropriate tag to each node. The JuJu Controller needs to be tagged “juju” and the nodes that will be used as compute nodes need to be tagged “compute”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

With nodes renamed, tagged and in “New” state, the list of Machines on the MAAS web GUI should look something like this.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

Now the machines are ready to be “Commissioned”. To do this select all the machines and click on the green “Take Action” button, then click on “Commission”, and “Start commissioning machines”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }

This process will take some time but once the machines are in a “Ready” state you can work on deploying the JuJu Controller and Compute nodes.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


# [Next Section: JuJu](https://bsu-cybersecurity.github.io/posts/openstack-deployment-juju/)
