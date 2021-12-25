---
layout: post
title: "OpenStack Deployment - JuJu"
date: 2021-01-26 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, charms, juju, maas]
---
JuJu will have two parts to it, one is the application we interact with through command-line (which gets installed on the MAAS Controller), and the other is the JuJu Controller itself which actually “orchestrates” the commands we give JuJu through the command line. JuJu will have control of MAAS, so it can power-cycle nodes as needed when we deploy the OpenStack Charm containers later on.
# Installation
As mentioned, we need to install the JuJu application on the MAAS Controller. As of the writing of this guide the latest version of JuJu that played “nicely” with OpenStack Wallaby is JuJu 2.8.10. We had some issues deploying OpenStack charms using JuJu 2.9, however this may have been patched by the time this guide is being used.

Use the following command to install JuJu 2.8.10

```bash
sudo snap install juju --channel=2.8/stable --classic
```
# Configuration
At this point, it is important to note that **JuJu commands should not be run as root**. JuJu keeps track of permissions and owners of configuration files, and if these do not match perfectly JuJu will throw error messages.

With that being said, the first step of the JuJu configuration is to create a cloud. This is done by first creating a YAML file called **maas-cloud.yaml** specifying the cloud configurations as follows, where “maas-cloud” is the name of the cloud (replace the IP **10.0.0.2** with that of the MAAS GUI).

```yaml
clouds:
  maas-cloud:
    type: maas
    auth-types: [oauth1]
    endpoint: http://10.0.0.2:5240/MAAS
```

Then use the following JuJu command to create a cloud using the configurations specified in the **maas-cloud.yaml** and specify the name of the cloud, in this case “maas-cloud”.

```bash
juju add-cloud --client -f maas-cloud.yaml maas-cloud
```
Now you need to follow similar steps to add MAAS credentials (API key) to the “maas-cloud” that was just created, so that JuJu can control MAAS. This is done by first creating a file called **maas-cred.yaml** with the following specifications, where “admin” is the name you want to assign to the user credentials (API key).

```yaml
credentials:
  maas-cloud:
    admin:
      auth-type: oauth1
      maas-oauth: PCwrBEpRsbcPu9FDzT:5YrGQWLs45RpsgHD4R:JvW9ywBqCS6SDZHzVW9g6yAVNYA6BNXn
```


The “**maas-auth**” API key above is found in the “Controller” tab on the MAAS web GUI, under “API Keys”.

![Shadow Avatar](https://cdn.jsdelivr.net/gh/cotes2020/chirpy-images/posts/20190808/window.png){: .shadow width="90%" }


With the **maas-creds.yaml** finished, we need to add the credentials to the “maas-cloud” we created in the step before. To do this use the following command.

```bash
juju add-credential --client -f maas-creds.yaml maas-cloud
```

At this point you’ve created a MAAS cloud that JuJu can control called “**maas-cloud**”, and you can now deploy the JuJu Controller Node.

# Deployment
There are two simple steps to deploying the JuJu Controller node.

First, we need to bootstrap a node using the following command, where we are specifying that we want Ubuntu Focal, and we tell juju to only bootstrap nodes tagged “juju”. We then specify the “**maas-cloud**” where the node lives, and we name the controller we are creating “juju-controller”.

```bash
juju bootstrap --bootstrap-series=focal --constraints tags=juju maas-cloud juju-controller
```

If you followed all steps correctly, the node you tagged “juju” within MAAS will be powered on and Ubuntu 20.04 will be deployed on this node. The deployment process will take some time, you will know youre ready for the last step of the JuJu Controller deployment when the command line output looks something like this.

```bash
juju bootstrap --bootstrap-series=focal --constraints tags=juju maas-cloud juju-controller

Creating Juju controller "juju-controller" on maas-cloud
Looking for packaged Juju agent version 2.8.10 for amd64
Launching controller instance(s) on maas-cloud...
- 4cw3ee (arch=amd64 mem=7G cores=16)
  Installing Juju agent on bootstrap instance
  Fetching Juju Dashboard 0.6.2
  Waiting for address
  Attempting to connect to 192.168.1.240:22
  Connected to 192.168.1.240
  Running machine configuration script...
  Bootstrap agent now started
  Contacting Juju controller at 192.168.1.240 to verify accessibility...

Bootstrap complete, controller "juju-controller" is now available
Controller machines are in the "controller" model
Initial model "default" added
```

The last step is for organization purposes. We will create what JuJu calls a “model”. All the OpenStack containers we deploy later and their relationships will live within that “model”. In this case we are calling the model “openstack”.

```bash
juju add-model --config default-series=focal openstack
```

To check if JuJu was deployed correctly input the JuJu status command.

```bash
juju status
```

The output should look like this.

```bash
juju status

Model      Controller       Cloud/Region  Version  SLA          Timestamp
openstack  juju-controller  maas-cloud    2.8.10   unsupported  02:09:10Z

Model "admin/openstack" is empty.
```
With the model created, you are now ready to begin deploying OpenStack.


# [Next Section: Charms](https://bsu-cybersecurity.github.io/posts/openstack-deployment-charms/)
