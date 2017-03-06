Vagrant Virtualbox setup for CS Engine 1.13.1-cs2, UCP 2.1.0 and DTR 2.2.2 with NFS
========================

The following set of instructions helps Docker DataCenter across multiple vms with static ip addresses:

* NFS Server node - `nfs-server-node` on 172.28.128.20
* HAProxy node - `haproxy-node` on 172.278.128.21
* Manager node - UCP controller `ucp-nfs-node1` on 172.28.128.22
* Worker node - DTR replica, `dtr-nfs-node1` on 172.28.128.23
* Worker node - DTR replica `dtr-nfs-node2` on 172.28.128.24
* Worker node - DTR replica, `dtr-nfs-node3` on 172.28.128.25

UCP traffic is proxied from `https://ucp.local` to 172.28.128.22. DTR traffic is load balanced from `https://dtr.local` to 172.28.128.23, 172.28.128.24, and 172.28.128.25. 

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

## Point ucp.local and dtr.local to HAProxy ip addresses
```
# edit /etc/hosts on macOS
sudo vi /private/etc/hosts
# add this line
172.28.128.21   dtr.local ucp.local
# save hosts config
sudo killall -HUP mDNSResponder
```

## Bring up/Resume NFS, UCP and DTR nodes
Make sure to bring up `nfs-server-node` prior to bringing up the DTR nodes, since DTR relies on NFS for registry image storage.
```
vagrant up nfs-server-node haproxy-node ucp-nfs-node1 dtr-nfs-node1 dtr-nfs-node2 dtr-nfs-node3
```

## Push sample image to DTR to test NFS

Pull redis image, tag it, and then push to DTR
```
$ docker pull redis
Using default tag: latest
latest: Pulling from library/redis
693502eb7dfb: Pull complete
338a71333959: Pull complete
83f12ff60ff1: Pull complete
4b7726832aec: Pull complete
19a7e34366a6: Pull complete
622732cddc34: Pull complete
3b281f2bcae3: Pull complete
Digest: sha256:4c8fb09e8d634ab823b1c125e64f0e1ceaf216025aa38283ea1b42997f1e8059
Status: Downloaded newer image for redis:latest
$ docker tag redis 172.28.128.22/engineering/redis
$ docker login 172.28.128.22 -u admin
Login Succeeded
$ docker push 172.28.128.22/engineering/redis
The push refers to a repository [172.28.128.22/engineering/redis]
450fce7b3b76: Pushed
9444719e7966: Pushed
61178e9e9bce: Pushed
c235d5b4caa3: Pushed
307248831aca: Pushed
387483b2c715: Pushed
a2ae92ffcd29: Pushed
latest: digest: sha256:dc458aa3ba58a59532663abdc199c84c1d3bbd7379bf91e6b4b9ed61b34a0bb4 size: 1783
```

Check that repository exists

```
$ vagrant ssh nfs-server-node
$ ls -alF /var/nfs/dtr/docker/registry/v2/repositories/engineering/redis/
total 20
drwxr-xr-x 5 nobody nogroup 4096 Mar  2 21:54 ./
drwxr-xr-x 3 nobody nogroup 4096 Mar  2 21:54 ../
drwxr-xr-x 3 nobody nogroup 4096 Mar  2 21:54 _layers/
drwxr-xr-x 4 nobody nogroup 4096 Mar  2 21:54 _manifests/
drwxr-xr-x 2 nobody nogroup 4096 Mar  2 21:54 _uploads/
```

## Pause UCP, DTR, and NFS nodes

```
vagrant suspend dtr-nfs-node3 dtr-nfs-node2 dtr-nfs-node1 ucp-nfs-node1 haproxy-node nfs-server-node
```

## Resume UCP, DTR, and NFS nodes

```
vagrant resume nfs-server-node haproxy-node ucp-nfs-node1 dtr-nfs-node1 dtr-nfs-node2 dtr-nfs-node3
```

## Destroy UCP, DTR, and NFS nodes

```
vagrant destroy dtr-nfs-node3 dtr-nfs-node2 dtr-nfs-node1 ucp-nfs-node1 haproxy-node nfs-server-node
```
