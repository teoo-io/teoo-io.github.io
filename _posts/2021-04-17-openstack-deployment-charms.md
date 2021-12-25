---
layout: post
title: "OpenStack Deployment - Charms"
date: 2021-01-25 09:00:00 -0500
categories: [Training-SOC, OpenStack]
tags: [openstack, cloud, deployment, soc, guide, charms, juju, maas]
---
We will be using JuJu to orchestrate the OpenStack deployment one individual “charm” container at a time. OpenStack can also be deployed via “bundles” - for instructions on a bundle deployment follow the official guide on the OpenStack website.

> “There are many moving parts involved in a charmed OpenStack install. During much of the process there will be components that have not yet been satisfied, which will cause error-like messages to be displayed in the output of the juju status command. Do not be alarmed. Indeed, these are opportunities to learn about the interdependencies of the various pieces of software. Messages such as Missing relation and blocked will vanish once the appropriate applications and relations have been added and processed.”

# Installation
First, make sure you are interacting with the correct juju controller and model.

```bash
juju switch juju-controller:openstack
```

With that out of the way, it may be useful to monitor the deployment progress either through the JuJu dashboard or using the JuJu status command.
### Monitoring the OpenStack Deployment
To access the JuJu dashboard, enter the following command.

```bash
juju dashboard
```

The output will look something like this.
```bash
juju dashboard
  Dashboard 0.6.2 for controller "juju-controller" is enabled at:
  https://192.168.1.240:17070/dashboard
  Your login credential is:
  username: admin
  password: 536d568fb04609daee40c61abc36844d
```
Alternatively, to monitor JuJu status use this command.

```bash
watch -n 1 -c juju status --color
```

## Ceph-OSD
The first charm we will deploy is the Ceph-osd charm or “object storage device”. This is done by first creating a YAML file called **ceph-osd.yaml** which specifies the charm’s configurations, where “**/dev/sdb**” is the path to the ```sdb```drive.

```yaml
ceph-osd:
  osd-devices: /dev/sdb
  source: cloud:focal-wallaby
```

Next, we will deploy the Ceph-osd charm to 4 nodes. Since we only have compute nodes, the “compute” tag constraint is not necessary strictly speaking, but it is good practice for future deployments.

```bash
juju deploy -n 4 --config ceph-osd.yaml --constraints tags=compute ceph-osd
```

With this being the first charm deployed to the compute nodes, MAAS will first power on the servers and deploy Ubuntu 20.04 to each node - and once that is done, JuJu will proceed to install the charm software. You can monitor the progress using the JuJu status command output - it will look like this once the Ceph-osd charm is finished installing.

```bash
Model      Controller       Cloud/Region  Version  SLA          Timestamp
openstack  juju-controller  maas-cloud    2.8.10   unsupported  05:09:54Z

App       Version  Status   Scale  Charm     Store       Rev  OS      Notes
ceph-osd  16.2.0   blocked      4  ceph-osd  jujucharms  310  ubuntu

Unit         Workload  Agent  Machine  Public address  Ports  Message
ceph-osd/0  blocked  idle  0  192.168.1.241  Missing relation: monitor
ceph-osd/1* blocked  idle  1  192.168.1.235  Missing relation: monitor
ceph-osd/2  blocked  idle  2  192.168.1.236  Missing relation: monitor
ceph-osd/3  blocked  idle  3  192.168.1.237  Missing relation: monitor

Machine  State    DNS            Inst id  Series  AZ       Message
0        started  192.168.1.241  node0  focal   default  Deployed
1        started  192.168.1.235  node1  focal   default  Deployed
2        started  192.168.1.236  node2  focal   default  Deployed
3        started  192.168.1.237  node3  focal   default  Deployed
```

## Nova Compute
With the Ceph-osd installed, the Nova Compute charm is next on the list of deployment. The process is very similar across charms. First, create the file **nova-compute.yaml**.

```yaml
nova-compute:
  config-flags: default_ephemeral_format=ext4
  enable-live-migration: true
  enable-resize: true
  migration-auth-type: ssh
  openstack-origin: cloud:focal-wallaby
```

Then, deploy the Nova Compute charm using the file **nova-compute.yaml** configurations.

```bash
juju deploy -n 3 --to 1,2,3 --config nova-compute.yaml nova-compute
```

## MySQL InnoDB Cluster
```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 mysql-innodb-cluster
```

## Vault
```bash
juju deploy --to lxd:3 vault
```

```bash
juju deploy mysql-router vault-mysql-router
juju add-relation vault-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation vault-mysql-router:shared-db vault:shared-db
```


### Unseal Vault
To unseal the Vault container, first install the vault agent application on the MAAS Controller.

```bash
sudo snap install vault
```
With the Vault application installed, run ```juju status``` to obtain the IP of the vault container.
> **Hint:** The port specified on ```juju status``` for this IP is 8200. This step is to tell the Vault application where to reach the Vault container.


```bash
export VAULT_ADDR="http://10.0.0.126:8200"
```

Now ask the vault for 5 keys.

```bash
vault operator init -key-shares=5 -key-threshold=3
```
Sample output:

```bash
Unseal Key 1: XONSc5Ku8HJu+ix/zbzWhMvDTiPpwWX0W1X/e/J1Xixv
Unseal Key 2: J/fQCPvDeMFJT3WprfPy17gwvyPxcvf+GV751fTHUoN/
Unseal Key 3: +bRfX5HMISegsODqNZxvNcupQp/kYQuhsQ2XA+GamjY4
Unseal Key 4: FMRTPJwzykgXFQOl2XTupw2lfgLOXbbIep9wgi9jQ2ls
Unseal Key 5: 7rrxiIVQQWbDTJPMsqrZDKftD6JxJi6vFOlyC0KSabDB

Initial Root Token: s.ezlJjFw8ZDZO6KbkAkm605Qv

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 3 key to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```

Now run ```vault operator unseal $Key``` and replace ```$Key``` with each of the 5 keys above.

```bash
vault operator unseal XONSc5Ku8HJu+ix/zbzWhMvDTiPpwWX0W1X/e/J1Xixv
vault operator unseal J/fQCPvDeMFJT3WprfPy17gwvyPxcvf+GV751fTHUoN/
vault operator unseal +bRfX5HMISegsODqNZxvNcupQp/kYQuhsQ2XA+GamjY4
vault operator unseal FMRTPJwzykgXFQOl2XTupw2lfgLOXbbIep9wgi9jQ2ls
vault operator unseal 7rrxiIVQQWbDTJPMsqrZDKftD6JxJi6vFOlyC0KSabDB
```

Now grab the token (line 7 in the sample output above), and enter these commands:

> **Note:** Replace the token below ```s.ezlJjFw8ZDZO6KbkAkm605Qv``` with your own.


```bash
export VAULT_TOKEN=s.ezlJjFw8ZDZO6KbkAkm605Qv
vault token create -ttl=10m
```

Sample output:

```bash
Key                  Value
---                  -----
token                s.QMhaOED3UGQ4MeH3fmGOpNED
token_accessor       nApB972Dp2lnTTIF5VXQqnnb
token_duration       10m
token_renewable      true
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

Now grab the last token (line 3 of the sample output above), and enter this command:

> **Note:** Replace the token below ```s.QMhaOED3UGQ4MeH3fmGOpNED``` with your own.

```bash
juju run-action --wait vault/leader authorize-charm token=s.QMhaOED3UGQ4MeH3fmGOpNED
```

### Generate the Certificate Authority (CA)
The last step to unseal the vault is to generate a CA using the command below.

```bash
juju run-action --wait vault/leader generate-root-ca
```

At this point the Vault is ready, and you can continue with the charm deployment.

```bash
juju add-relation mysql-innodb-cluster:certificates vault:certificates
```

## Neutron networking
Create a file called **neutron.yaml** with the YAML below.

> **Note:** The NIC in this case is called eth1 (line2 below) - but more than likely this will be different for our environment.

> **Tip:** To find out the name of the NIC, go in the MAAS web GUI and click on 'machines' at the top. Now click on one of the 'compute' nodes and click on 'Networking', this will give you a list of the NIC ports and what they are called. There is one NIC that has a green checkmark saying it is used for PXE booting. We need one that says 'Unconfigured'. If this 'unconfigured' NIC is called "eno2", you would replace "eth1" on the YAML below with "eno2" (Example: ```bridge-interface-mappings: br-ex:eno2```).


```yaml
ovn-chassis:
  bridge-interface-mappings: br-ex:eth1
  ovn-bridge-mappings: physnet1:br-ex
neutron-api:
  neutron-security-groups: true
  flat-network-providers: physnet1
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
ovn-central:
  source: cloud:focal-wallaby
```

```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --config neutron.yaml ovn-central
```

```bash
juju deploy --to lxd:1 --config neutron.yaml neutron-api
```

```bash
juju deploy neutron-api-plugin-ovn
juju deploy --config neutron.yaml ovn-chassis
```

```bash
juju add-relation neutron-api-plugin-ovn:neutron-plugin neutron-api:neutron-plugin-api-subordinate
juju add-relation neutron-api-plugin-ovn:ovsdb-cms ovn-central:ovsdb-cms
juju add-relation ovn-chassis:ovsdb ovn-central:ovsdb
juju add-relation ovn-chassis:nova-compute nova-compute:neutron-plugin
juju add-relation neutron-api:certificates vault:certificates
juju add-relation neutron-api-plugin-ovn:certificates vault:certificates
juju add-relation ovn-central:certificates vault:certificates
juju add-relation ovn-chassis:certificates vault:certificates
```

```bash
juju deploy mysql-router neutron-api-mysql-router
juju add-relation neutron-api-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation neutron-api-mysql-router:shared-db neutron-api:shared-db
```

## Keystone
Create a file called **keystone.yaml** with the YAML below.

```yaml
keystone:
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:0 --config keystone.yaml keystone
```

```bash
juju deploy mysql-router keystone-mysql-router
juju add-relation keystone-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation keystone-mysql-router:shared-db keystone:shared-db
```

```bash
juju add-relation keystone:identity-service neutron-api:identity-service
juju add-relation keystone:certificates vault:certificates
```

## RabbitMQ

```bash
juju deploy --to lxd:2 rabbitmq-server
```

```bash
juju add-relation rabbitmq-server:amqp neutron-api:amqp
juju add-relation rabbitmq-server:amqp nova-compute:amqp
```

```bash
[SAMPLE JUJU STATUS OUTPUT GOES HERE]
```
## Nova cloud controller
Create a file called **nova-cloud-controller.yaml** with the YAML below.

```yaml
nova-cloud-controller:
  network-manager: Neutron
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```
```bash
juju deploy --to lxd:3 --config nova-cloud-controller.yaml nova-cloud-controller
```
```bash
juju deploy mysql-router ncc-mysql-router
juju add-relation ncc-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation ncc-mysql-router:shared-db nova-cloud-controller:shared-db
```
```bash
juju add-relation nova-cloud-controller:identity-service keystone:identity-service
juju add-relation nova-cloud-controller:amqp rabbitmq-server:amqp
juju add-relation nova-cloud-controller:neutron-api neutron-api:neutron-api
juju add-relation nova-cloud-controller:cloud-compute nova-compute:cloud-compute
juju add-relation nova-cloud-controller:certificates vault:certificates
```


## Placement
Create a file called **placement.yaml** with the YAML below.

```yaml
placement:
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:3 --config placement.yaml placement
```
```bash
juju deploy mysql-router placement-mysql-router
juju add-relation placement-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation placement-mysql-router:shared-db placement:shared-db
```
```bash
juju add-relation placement:identity-service keystone:identity-service
juju add-relation placement:placement nova-cloud-controller:placement
juju add-relation placement:certificates vault:certificates
```

## OpenStack dashboard
```bash
juju deploy --to lxd:2 --config openstack-origin=cloud:focal-wallaby openstack-dashboard
```
```bash
juju deploy mysql-router dashboard-mysql-router
juju add-relation dashboard-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation dashboard-mysql-router:shared-db openstack-dashboard:shared-db
```
```bash
juju add-relation openstack-dashboard:identity-service keystone:identity-service
juju add-relation openstack-dashboard:certificates vault:certificates
```
## Glance
Create a file called **glance.yaml** with the YAML below.

```yaml
glance:
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```
```bash
juju deploy --to lxd:3 --config glance.yaml glance
```
```bash
juju deploy mysql-router glance-mysql-router
juju add-relation glance-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation glance-mysql-router:shared-db glance:shared-db
```
```bash
juju add-relation glance:image-service nova-cloud-controller:image-service
juju add-relation glance:image-service nova-compute:image-service
juju add-relation glance:identity-service keystone:identity-service
juju add-relation glance:certificates vault:certificates
```
```bash
[SAMPLE JUJU STATUS OUTPUT GOES HERE]
```

## Ceph monitor
Create a file called **ceph-mon.yaml** with the YAML below.

```yaml
ceph-mon:
  expected-osd-count: 3
  monitor-count: 3
  source: cloud:focal-wallaby
```
```bash
juju deploy -n 3 --to lxd:0,lxd:1,lxd:2 --config ceph-mon.yaml ceph-mon
```
```bash
juju add-relation ceph-mon:osd ceph-osd:mon
juju add-relation ceph-mon:client nova-compute:ceph
juju add-relation ceph-mon:client glance:ceph
```

## Cinder
Create a file called **cinder.yaml** with the YAML below.

```yaml
cinder:
  block-device: None
  glance-api-version: 2
  worker-multiplier: 0.25
  openstack-origin: cloud:focal-wallaby
```

```bash
juju deploy --to lxd:1 --config cinder.yaml cinder
```
```bash
juju deploy mysql-router cinder-mysql-router
juju add-relation cinder-mysql-router:db-router mysql-innodb-cluster:db-router
juju add-relation cinder-mysql-router:shared-db cinder:shared-db
```
```bash
juju add-relation cinder:cinder-volume-service nova-cloud-controller:cinder-volume-service
juju add-relation cinder:identity-service keystone:identity-service
juju add-relation cinder:amqp rabbitmq-server:amqp
juju add-relation cinder:image-service glance:image-service
juju add-relation cinder:certificates vault:certificates
```
```bash
juju deploy cinder-ceph
```
```bash
juju add-relation cinder-ceph:storage-backend cinder:storage-backend
juju add-relation cinder-ceph:ceph ceph-mon:client
juju add-relation cinder-ceph:ceph-access nova-compute:ceph-access
```
## Ceph RADOS Gateway
```bash
juju deploy --to lxd:0 --config source=cloud:focal-wallaby ceph-radosgw
```
```bash
juju add-relation ceph-radosgw:mon ceph-mon:radosgw
```
## NTP
```bash
juju deploy ntp
```
```bash
juju add-relation ceph-osd:juju-info ntp:juju-info
```

## Final results and dashboard access
Once the ```juju status``` output has settled it should look something like this:
```bash
[SAMPLE JUJU STATUS OUTPUT GOES HERE]
```

At this point you should have a fully deployed OpenStack. To retrieve the OpenStack Dashboard IP enter the following command:
```bash
juju status --format=yaml openstack-dashboard | grep public-address | awk '{print $2}' | head -1
```

And to retrieve the password use:

```bash
juju run --unit keystone/leader leader-get admin_passwd
```

### Make Accessing the Openstack Dashboard easier
It is sometimes helpful to have aliases for commonly used commands.

To create an alias that outputs the URL and password of the OpenStack Dashbaord, use your favorite text editor to edit the ```.bashrc``` file.

```bash
vim ~/.bashrc
```

With the ```.bashrc``` file open, scroll down to where you see aliases listed. Here enter the following aliases in new lines:

```bash
alias openstack-ip='juju status --format=yaml openstack-dashboard | grep public-address | awk '"'"'{print $2}'"'"' | head -1'
alias openstack-pass='juju run --unit keystone/leader leader-get admin_passwd'
alias openstack-login='openstackip=$(openstack-ip) && echo "Dashboard URL: https://"$openstackip"/horizon" && echo "Username: admin" && openstackpass=$(openstack-pass) && echo "Passowrd: "$openstackpass && echo "Domain: admin_domain"'
```

Now save your edits, and exit the file. To make the new changes effective source the ```.bashrc``` file.
```bash
source ~/.bashrc
```

If you followed the steps correctly the command ```openstack-login``` should output your Dashboard URL, username, password, and domain.



## VM consoles
```bash
juju config nova-cloud-controller console-access-protocol=novnc
```

# [Next Section: Configuration](https://bsu-cybersecurity.github.io/posts/openstack-deployment-configuration/)

