Vagrant Virtualbox setup for CS Engine 1.13.1-cs2, UCP 2.1.0 and DTR 2.2.2 with NFS
========================

The following set of instructions helps Docker DataCenter across multiple vms with static ip addresses:

* Manager node - UCP controller `ucp-nfs-node1` on 172.28.128.21
* Worker node - DTR replica, `dtr-nfs-node1` on 172.28.128.22
* Worker node - DTR replica `dtr-nfs-node2` on 172.28.128.23
* Worker node - DTR replica, `dtr-nfs-node3` on 172.28.128.24
* NFS Server - `nfs-server-node` on 172.28.128.20

## Download vagrant from Vagrant website

```
https://www.vagrantup.com/downloads.html
```

## Install Virtual Box

```
https://www.virtualbox.org/wiki/Downloads
```

## Download Xenial box
```
vagrant init ubuntu/xenial64
```

## Install Vagrant Plugins
```
vagrant plugin install vagrant-hostsupdater
vagrant plugin install vagrant-multiprovider-snap
```

## Bring up/Resume UCP, DTR, and NFS nodes

```
vagrant up nfs-server-node ucp-nfs-node1 dtr-nfs-node1 dtr-nfs-node2 dtr-nfs-node3
```

## Stop UCP, DTR, and NFS nodes

```
vagrant halt nfs-server-node ucp-nfs-node1 dtr-nfs-node1 dtr-nfs-node2 dtr-nfs-node3
```

## Destroy UCP, DTR, and NFS nodes

```
vagrant destroy nfs-server-node ucp-nfs-node1 dtr-nfs-node1 dtr-nfs-node2 dtr-nfs-node3
```
