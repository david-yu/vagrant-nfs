Vagrant Virtualbox setup for CS Engine 1.13.1-cs1, UCP 2.1.0 and DTR 2.2.1 with NFS
========================

The following set of instructions helps Docker DataCenter across multiple vms with static ip addresses:

* UCP - ucp.local on 172.28.128.20
* DTR - dtr.local on 172.28.128.21
* Worker node - worker-node1 on 172.28.128.22
* Worker node - worker-node2 on 172.28.128.23

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

## Bring up/Resume UCP, DTR, and Jenkins nodes

```
vagrant up ucp-vancouver-node1 dtr-vancouver-node1 worker-node1 worker-node2
```

## Stop UCP, DTR, and Jenkins nodes

```
vagrant halt ucp-vancouver-node1 dtr-vancouver-node1 worker-node1 worker-node2
```

## Destroy UCP, DTR, and Jenkins nodes

```
vagrant destroy ucp-vancouver-node1 dtr-vancouver-node1 worker-node1 worker-node2
```
