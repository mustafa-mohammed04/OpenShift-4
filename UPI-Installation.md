## OpenShift-Cluster "UPI-Installation"
## IPXE
## BootStrap Node "OS-realase is RHEL"
## Bastion-Host OR Jump-Host for Cluster "OS-realase is RHEL"
## Note: as a Best Practice Control-node "odd Number" at Least 3 Controlplane-nodes, Worker-node "even-number" at least 2 Worker-nodes   "OS-realase is core-os"
### On Bootstrap-Node make that:
## Install PreReq Packages and copy
``` bash
yum install httpd
yum install dnsmasq -y
yum install syslinux-tftpboot.noarch -y
```
## Download  IPXE Package "install Package" and copy kpxe to cation where files are stored for clients "tftpboot"
default path: /var/lib/tftpboot
``` bash
wget http://boot.ipxe.org/undionly.kpxe
cp undionly.kpxe /tftpboot/
```
## Configure DHCP or Add DHCP Configuration
``` bash
cd /etc/dnsmasq.d/
vim dhcpd.conf
#global options
domain-needed
bogus-priv
no-resolv
filterwin2k
expand-hosts
dhcp-authoritative
dhcp-ignore=tag:!known
except-interface=lo
log-queries
log-dhcp

domain=hqdomain.com
interface=ens3,ens7

dhcp-range=<-IP Range From->,<-IP Range TO->,255.255.255.0,12h
dhcp-option=option:router,<Gateway IP>
dhcp-option=option:dns-server,<-IP OF DNS1->,<-IP OF DNS1->

#tftp
enable-tftp
tftp-root=/tftpboot

#IPXE
dhcp-boot=pxelinux.0
dhcp-boot=undionly.kpxe
dhcp-match=set:ipxe,175
dhcp-boot=tag:ipxe,http://<-Help Node IP->/ocp.ipxe

dhcp-userclass=set:boot-ipxe,iPXE
dhcp-boot=tag:boot-ipxe,http://<-Help Node IP->/ocp.ipxe 
``` 




## Add DHCP Fixed IP for IPXE
``` bash
cd /etc/dnsmasq.d/
vim dhcp-hosts
dhcp-host=<-Mac Address->,<-Node PXI Temp IP->,<DNS Record>
dhcp-host=<-Mac Address->,<-Node PXI Temp IP->,<DNS Record>
## repeat this for all cluster node
``` 

## Download needed image from redhat  "cloud.redhat.com"
rhcos-<version>-live-initramfs.x86_64.img
rhcos-<version>-live-kernel-x86_64
rhcos-<version>-live-rootfs.x86_64.img

## Add Node image , ignition files and network configuration 
``` bash
cd /var/www/html/
vim ocp.ipxe
#!ipxe

set boot-url http://172.16.155.51
set kernel rhcos-4.10.16-x86_64-live-kernel-x86_64
set root-fs rhcos-4.10.16-x86_64-live-rootfs.x86_64.img
set initramfs rhcos-4.10.16-x86_64-live-initramfs.x86_64.img

:start
menu iPXE boot menu for Openshift
item Bootstrap Bootstrap Node
item Master1    Master1 Node
item Master2    Masterx Node
##add all needed node
item reboot    Reboot
item exit      Exit (boot local disk)
choose selected && goto ${selected}

:reboot
reboot

:exit
exit

:Bootstrap
echo Booting Bootstrap Node

#set iface1 ip::GW:netmask:hostname:iface:none

kernel ${boot-url}/${kernel} initrd=main coreos.live.rootfs_url=${boot-url}/${root-fs} coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=${boot-url}/bootstrap.ign rd.neednet=1 ip=172.16.155.52::172.16.155.1:255.255.255.0:bootstrap.hub-ocp-prod.hqdomain.com:ens3:none nameserver=172.16.20.20 nameserver=172.16.20.21
initrd --name main ${boot-url}/${initramfs}
boot


################################

:Master1
echo Booting Master1 Node

#set iface1 ip::GW:netmask:hostname:iface:none

kernel ${boot-url}/${kernel} initrd=main coreos.live.rootfs_url=${boot-url}/${root-fs} coreos.inst.install_dev=/dev/sda coreos.inst.ignition_url=${boot-url}/master.ign rd.neednet=1 ip=172.16.155.11::172.16.155.1:255.255.255.0:control-plane-1.hub-ocp-prod.hqdomain.com:ens3:none nameserver=172.16.20.20 nameserver=172.16.20.21
initrd --name main ${boot-url}/${initramfs}
boot

################################



```



add these images in to /var/www/html/

## Download OpenShift installation file and copy these files.tar.gz under 
openshift-install-linux-<version>.tar.gz
openshift-client-linux-<version>.tar.gz

``` bash
mkdir ~/ocp4/installation
```

## Extract Donwloaded files.tar.gz under pervious dir
``` bash
cd ~/ocp4/installation
tar xvf openshift-install-linux-4.10.24.tar.gz
tar xvf  openshift-client-linux-4.10.24.tar.gz
```

## Copy Client tools to /usr/local/bin and Make auto-completion 
``` bash
cp openshift-install kubectl oc /usr/local/bin
oc completion bash > oc_bash_completion
cp oc_bash_completion /etc/bash_completion.d/
openshift-install completion bash > openshift_bash_completion
cp openshift_bash_completion /etc/bash_completion.d/

```

## Add auto-completion in ~/.bashrc and add line end this file
``` bash
source /etc/bash_completion.d/oc_bash_completion
```

## Download pull secret   "cloud.redhat.com  >   console login  "
## Create Key-Generate of ssh to user
``` bash
ssh-keygen -t <type>
```
## create yaml file for installation
``` bash
cd ~
vim install-config.yaml

apiVersion: v1
baseDomain: ocp.lan
compute:
  - hyperthreading: Enabled
    name: worker
    replicas: 0 # Must be set to 0 for User Provisioned Installation as worker nodes will be manually deployed.
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: lab # Cluster name
networking:
  clusterNetwork:
    - cidr: <cidr/perfix>
      hostPrefix: <num-of hostPre>
  networkType: OpenShiftSDN
  serviceNetwork:
    - <service-ip/prefix>
platform:
  none: {}
fips: false
pullSecret: '{"auths": ...}'
sshKey: "ssh-ed25519 AAAA..."
```
## Copy yaml file to new directory
``` bash
mkdir ocp4/installation/installation_files/
cp install-config.yaml ocp4/installation/installation_files/
cd ocp4/installation/installation_files/
```
## Create Manifest files 
``` bash
openshift-install create manifests
vim manifests/XXschedulableX.yaml

```
## Note: Edit schedulable.yaml	false 			to make pod assign or add only on workers nodes

## Create Ignition files  and copy files under http path
``` bash
openshift-install create ignition-configs
cp *.ign /var/www/html/
chcon -R -t httpd_sys_content_t /var/www/html/
chown -R apache: /var/www/html/
chmod 755 /var/www/html/
```

## Restart All Services that used
``` bash
systemctl restart httpd -y
systemctl restart dnsmasq -y 

```

## Disable and stop Firewall service 

``` bash
systemctl diable  --now firewalld
systemctl status firewalld
```

## In Order Make that
1- Start boot from BootStrap-node
2- Start Boot Controlplane-nodes  / Master-Nodes
3- Start Boot Worker-nodes

## Export kubeconfig file 
``` bash
export KUBECONFIG=~/ocp4/installation/installation_files/auth/kubeconfig
openshift-install --dir ~/ocp4/installation/installation_files/ wait-for bootstrap-complete --log-level=debug
openshift-install --dir ~/ocp4/installation/installation_files/  wait-for install-complete
from bootstrap:
journalctl -b -f -u release-image.service -u bootkube.service
```
## Show CSR and accept all csr using Script
``` bash
while true ; do oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve ; done
```

## show api server
``` bash
oc whoami --show-server
https://<api-hostname-FQN>:6443
```
## To Login Cluster as GUI "Console"
``` bash
oc whoami --show-console
```
## Login as kubeadmin user in cli or console
## Default password of kubeadmin ""
## If you lose access to the kubeadmin password, you may need to re-install the cluster 
## Note after login make htpasswd user and ldap user and delete kubeadmin user and path of password
## cat /home/user/ocp-install/auth/kubeadmin-password

## Install  OperatorHub
1- Login cluster as console
2- Adminstrators > Operators > OperatorHub > <any-Operator>

## Create htpasswd  User
1- Locate the HTPasswd File  "Check the file location by running"
``` bash
oc get oauth cluster -o yaml | grep fileData
```
2- Extract the current htpasswd file from the Kubernetes secret
``` bash
oc extract secret/htpasswd-secret -n openshift-config --to=-
```
3- Save the content as htpasswd

``` bash
oc extract secret/htpasswd-secret -n openshift-config --to=. --confirm
```
4- Add the User to the HTPasswd File "Use the htpasswd tool to add a new user:"
``` bash
htpasswd -B -b htpasswd <username> <password>
```
5- Update the htpasswd file in the Kubernetes secret
``` bash
oc create secret generic htpasswd-secret --from-file=htpasswd=htpasswd -n openshift-config --dry-run=client -o yaml | oc apply -f -
```
6- Test the User
``` bash
oc login -u <username> -p <password> <api-url>
```
7- Ensure the new user appears in the cluster and after login in console 
``` bash
oc get users
oc get identities.user.openshift.io
```

8- Assign roles to the user  "Optional" as CLI
``` bash
oc adm policy add-cluster-role-to-user cluster-admin <username>
```
8- Assign roles to the user  "Optional" as Console
Adminstrator > User Management > Username > RoleBindings > Create Binding >  Role name "cluster-admin"

## Create Ldap user as CLI
## Note i have Group "DevOps-Main" and under this group list of users "devops.mustafa"
## You have 2 Important files "ldap-sync.yaml & Ldap-Group-Whitelist"
``` bash
vim ldap-sync.yaml
kind: LDAPSyncConfig
apiVersion: v1
url: ldap://<dns-server>:389
insecure: <bool-value>
bindDN: "<>"
bindPassword: "<>"
rfc2307:
    groupsQuery:
        baseDN: "<>"
        scope: one
        derefAliases: never
        filter: (objectSid=*)
        pageSize: 0
    groupUIDAttribute: cn
    groupNameAttributes: [ name ]
    groupMembershipAttributes: [ member ]
    usersQuery:
        baseDN: "dc=hqdomain,dc=com"
        scope: sub
        derefAliases: never
        pageSize: 0
    userUIDAttribute: dn
    userNameAttributes: [ cn ]
    tolerateMemberNotFoundErrors: false
    tolerateMemberOutOfScopeErrors: false

```
``` bash
vim Ldap-Group-Whitelist
<gruop-name>
```
## Run the LDAP Sync Command 
``` bash
oc adm groups sync --sync-config=ldap-sync.yaml --whitelist=ldap-group-whitelist
oc adm groups sync --sync-config=ldap-sync.yaml --whitelist=ldap-group-whitelist --confirm
oc get groups
```

## Assign Roles to Synced Users as CLI
``` bash
oc adm policy add-role-to-group cluster-admin <group-name>
```
## Assign Roles to Synced Users as Console 
Adminstrator > User Management > Group-name > RoleBindings > Create Binding >  Role name "cluster-admin" 
cluster-admin is Full Access
if you want to read only Choose Role name "View"

## Create openshift-storage "recommend"
1- ODF OpenShift Data Foundation operator from OperatorHub
2- AWS EBS CSI Driver
3- NutaniCSI Driver 

## After Install any openshift-storage like ODF
1- List Available StorageClasses
``` bash
oc get storageclass
```
2- Set a Default StorageClass "Optional"
``` bash
oc annotate storageclass <storage-class-name> storageclass.kubernetes.io/is-default-class=true
```
3- Test PV Provisioning > Create test-pvc.yaml
``` bash
vim  test-pvc.yaml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: <your-storage-class>

```
4- Apply yaml file of PVC
``` bash
oc apply -f pvc-test.yaml

```

5- Check ODF Components
``` bash
oc get pods -n openshift-storage

```
6- Check CSI Driver Components
``` bash
oc get pods -n <csi-namespace>

```
