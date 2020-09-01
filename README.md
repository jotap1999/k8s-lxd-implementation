# LXC 

## LXD INIT

```bash
lxd init
	Would you like to use LXD clustering? (yes/no) [default=no]: 
	Do you want to configure a new storage pool? (yes/no) [default=yes]: 
	Name of the new storage pool [default=default]: 
	Name of the storage backend to use (btrfs, dir, lvm) [default=btrfs]: dir
	Would you like to connect to a MAAS server? (yes/no) [default=no]: 
	Would you like to create a new local network bridge? (yes/no) [default=yes]: 
	What should the new bridge be called? [default=lxdbr0]: 
	What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
	What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
	Would you like LXD to be available over the network? (yes/no) [default=no]: 
	Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
	Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```

## Create the k8s master and nodes

```bash
lxc launch ubuntu:18.04 kubermaster
lxc launch ubuntu:18.04 kubernode1
lxc launch ubuntu:18.04 kubernode2
```

## Edit each one of LXC container adding the following settings:
```bash
lxc config edit "nameContainer"
```

```bash
config:
  linux.kernel_modules: xt_conntrack,br_netfilter,ip_tables,ip6_tables,netlink_diag,nf_nat,overlay
  raw.lxc: "lxc.apparmor.profile=unconfined\nlxc.cap.drop= \nlxc.cgroup.devices.allow=a\nlxc.mount.auto=proc:rw
    sys:rw"
  security.privileged: "true"
  security.nesting: "true"

```

## Restart 
```bash
lxc exec kubermaster reboot
lxc exec kubernode1 reboot
lxc exec kubernode2 reboot
```

# Install and Set Up Kubernetes on the lxc

## 	Install Kubeadm and docker on all lxc containers (Master and Nodes)

```bash 
lxc exec "nameContainer" -- /bin/bash 
```
```bash 
curl -o- https://raw.githubusercontent.com/jotap1999/k8s-docker-Install-Script-Ubuntu/master/install.sh  | bash

echo 'L /dev/kmsg - - - - /dev/console' > /etc/tmpfiles.d/kmsg.conf

reboot
```

## 	Configure initial setup on Master Node. 

```bash
lxc exec kubermaster -- /bin/bash 
```

**Note:** If you want to configure the kuber with auth LDAP/SAMBA, head over to the [K8s Samba Authentication](https://github.com/jotap1999/k8s-samba-authentication) and ignore the following indications

```bash
kubeadm init --pod-network-cidr=10.244.0.0/16
	
	.....
	# the command below is necessary to run on Worker Node when he joins to the cluster, so remember it
	kubeadm join X.X.X.X:6443 --token nvr822.tjn09e85qw3a3vuz --discovery-token-ca-cert-hash sha256:866f645d9ec0da07f778b3c4abc4427e9967845d71add3252fbd691b86c0a9a7
```

```bash
# set cluster admin user
# if you set common user as cluster admin, login with it and run [sudo cp/chown ***]
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```bash
#Configure Pod Network with Flannel.
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
# show state (OK if STATUS = Ready)
kubectl get nodes

# show state (OK if all are Running)
kubectl get pods --all-namespaces
```

## 	Join in Kubernetes Cluster which is initialized on Master Node.

```bash
#kubernode1 and Kubernode2
lxc exec "nameContainerWorkerNode" -- /bin/bash 
```

```bash
#The command for joining is just the one [kubeadm join ***] which was shown on the bottom of the results on initial setup of Cluster
kubeadm join X.X.X.X:6443 --token nvr822.tjn09e85qw3a3vuz --discovery-token-ca-cert-hash sha256:866f645d9ec0da07f778b3c4abc4427e9967845d71add3252fbd691b86c0a9a7
```

```bash
#Verify Status on Master Node. It's Ok if all STATUS are Ready.
kubectl get nodes
```

#  Configure iptables on physical machine to forward the port 6443 (k8s) to the LXD container Kubermaster

```bash
iptables -t nat -A PREROUTING -p tcp --dport 6443  -j DNAT --to-destination ipKubermaster

#To save
apt -y install netfilter-persistent iptables-persistent
netfilter-persistent save
```

# References
- https://github.com/corneliusweig/kubernetes-lxd
- https://www.server-world.info/en/note?os=Ubuntu_18.04&p=kubernetes&f=3
