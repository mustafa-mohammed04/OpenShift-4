### Note
Memory at least: 4GB
vcpu: 2
storage: 20GB

## Install Hypervisor  "KVM and Virtualbox" and PreReq
``` bash
sudo apt-get update -y
sudo apt -y install qemu-kvm libvirt-dev bridge-utils libvirt-daemon-system libvirt-daemon virtinst bridge-utils libosinfo-bin libguestfs-tools virt-top virt-manager 
```
## Load and enable the module "vhost"
``` bash
sudo modprobe vhost_net
sudo lsmod | grep vhost
echo “vhost_net” | sudo tee -a /etc/modules

```

## Install KVM drivers and Dependencies "executable access"
``` bash
sudo usermod -a -G libvirt $(whoami)
newgrp libvirt || newgrp libvirtd
curl -L https://github.com/dhiltgen/docker-machine-kvm/releases/download/v0.10.0/docker-machine-driver-kvm-ubuntu16.04 -o docker-machine-driver-kvm
sudo mv docker-machine-driver-kvm /usr/local/bin/docker-machine-driver-kvm
sudo chmod +x /usr/local/bin/docker-machine-driver-kvm

```
## Start the default KVM network
``` bash
sudo virsh net-start default
sudo virsh net-autostart default
```

## Install Minishift-release
``` bash
export VER="1.34.3"
curl -L https://github.com/minishift/minishift/releases/<any-release> -o minishift-$VER-linux-amd64.tgz
tar xvf minishift-$VER-linux-amd64.tgz

```
OR Download release from https://github.com/minishift/minishift/releases/tag/v1.34.3 and extract file 

## Add Minishift binary to your $PATH environment variable
``` bash
sudo mv minishift-$VER-linux-amd64/minishift /usr/local/bin
```

## Confirm & Verify Installlation
``` bash
minishift version
```
## Start minishift
``` bash
minishift start
```

NOTE : Just in case you encounter error message Error starting the VM: Error configuring authorization on host: Too many retries waiting for SSH to be available. after did minishift start I solved it by these steps 

``` bash
#check if the VM is still running
sudo virsh list --all#if it still running, stop the VM
sudo virsh destroy minishift#delete the vm
sudo virsh undefine minishift#delete the folder ~/.minishift
rm -rf ~/.minishift#start Minishift again
minishift start

```

## check the server via web browser 
https://<ip-address>:8443/console                  is https://192.168.42.19:8443/console

## To login as administrator
Username: system
Password: admin


## Install Openshift Client Binary "oc"
``` bash
sudo cp ~/.minishift/cache/oc/<version>/linux/oc /usr/local/bin
```
## Check Version
``` bash
oc version
```
## Login to Cluster as CLI
``` bash
oc login -u system -p admin
```
## Login to Cluster as Console 
``` bash
oc whoami --show-console
```
and login in console same username: system and password: admin
