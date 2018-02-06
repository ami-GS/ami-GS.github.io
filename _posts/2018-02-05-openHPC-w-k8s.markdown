---
layout: post
title: "Use Kubernetes with OpenHPC(warewulf)"
categories: openhpc kubernetes
---

This post provide how-to use [kubernetes(k8s)](https://kubernetes.io/) with [OpenHPC](https://openhpc.community/)/[warewulf](https://warewulf.github.io/warewulf3/) environment.

OpenHPC is HPC environment configuration tool without doing it by hand. Currently it does not support k8s/container softwares. My motivation was to enable the feature by myself.

OpenHPC(warewulkf) enables doing PXE boot (boot from disk as well, but my focus is PXE boot). And several config files are distributed at provisioning time so that settings for k8s are needed.

If your compute nodes are booting from local disk (not using PXE boot), it would be better to reffer [official page](https://severalnines.com/blog/installing-kubernetes-cluster-minions-centos7-manage-pods-services)

## Configuration
My environment was as follows
- master node (192.168.201.10)
- compute nodes (192.168.201.100-131)

And use environment variables bellows are used in this article
``` shell
$ MIP=192.168.201.10 # master node's ip address
$ CIP=192.168.201.100 # compute node's ip address offset
$ NIP=192.168.201.5 # nfs node's ip address
$ CHROOT=/opt/ohpc/admin/images/centos7.2/
$ CNAME=compute # your compute node's name
$ CNUM=32 # num of computes
```

## Requirements
The requirements bellows are used in my environment. I believe other software versions would work as well.

- OpenHPC(warewulf) v1.3.3
- CentOS7.4 (both master and client nodes)
- Kubernetes

## Procedure
### Install on master
This is almost same as [This](https://severalnines.com/blog/installing-kubernetes-cluster-minions-centos7-manage-pods-services)
The difference is etcd location

``` shell
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"
```


### Before creating VNFS
In `recipe.sh` OpenHPC has problem to install kernel for compute nodes. We need to fix it at first

``` shell
$ yum -y --installroot=$CHROOT install kernel
$ # This installs latest kernel, but compute node uses base version
$ # and cause error when launching dockerd because of no ip_tables kernel module
$ # bellow could solve it
$ KERNEL_VERSION=`uname -r`
$ yum -y --installroot=$CHROOT install kernel-${KERNEL_VERSION%.*}
```

Then
``` shell
$ export OHPC_INPUT_LOCAL=./input.local # change config as you want
$ ./recipe.sh
```

Thre `./recipe.sh` tries to reboot compute node after finishing create VNFS, but you don't need to wait until then.

You can stop the process by Ctr-C just after finishing create VNFS.


### After creating VNFS
You could see the image directory, and you can re-create VNFS as you want using the directory like
``` shell
$ wwvnfs --chroot $CHROOT
```

### Setup config files
Let's setup `$CHROOT/etc/kubernetes/kubelet` config file.
``` shell
$ # because $CHROOT haven't install k8s yet
$ mkdir -p $CHROOT/etc/kubernetes/
$ # openHPC is using /opt/ohpc/pub/examples/ for template directory
$ mkdir -p /opt/ohpc/pub/examples/kubernetes/centos
$ cp /etc/kubernetes/kubelet /opt/ohpc/pub/examples/kubernetes/centos/kubelet.ww
$ # wwsh file import is for storing file in database
$ wwsh file import /opt/ohpc/pub/examples/kubernetes/centos/kubelet.ww
$ # wwsh file set is for aliasing filename and PATH
$ wwsh file set kubelet.ww --path=/etc/kubernetes/kubelet
$ # wwsh provision set is for letting warewulf know the config file to be distributed at provisioning time
$ wwsh provision set $CNAME* --fileadd=kubelet.ww
```
Setting for provisioning is done, next is to edit the file contents

``` shell
from: KUBELET_HOSTNAME="--hostname_override=127.0.0.1"
to:   KUBELET_HOSTNAME="--hostname_override=%{NETDEV::ETH0::IPADDR}"

from: KUBELET_API_SERVER="--api_servers=http://128.0.0.1:8080"
to:   KUBELET_API_SERVER="--api_servers=http://192.168.50.10:8080"
```
The `%{NETDEV::ETH0::IPADDR}` is for dynamic provisioning for each compute node.
You can use another interface, like `ETH1`, `IB0`. `IB0` is infiniband or omnipath interface.

### Install k8s for compute nodes
Install k8s and flannel to compute node via pdsh from master node
``` shell
$ pdsh -w $CNAME[1-$CNUM] yum install -y kubernetes flannel
$ pdsh -w $CNAME[1-$CNUM] perl -pi -e "s/127.0.0.1/$MIP/" /etc/sysconfig/flanneld
$ pdsh -w $CNAME[1-$CNUM] perl -pi -e "s/127.0.0.1/$MIP/" /etc/kubernetes/config
```
Because install k8s and flannel directly to `$CHROOT` cause error, that's why this installation is conducted after booting up the compute nodes.
