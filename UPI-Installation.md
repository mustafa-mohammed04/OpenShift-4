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

## Create keystone user
1- Ensure that OpenStack Keystone is running and configured properly
2- Obtain the Keystone service URL
3- Have Full access to your OpenShift cluster
4- Edit oauth
5- Create a Secret for the Client Secret
``` bash
oc edit oauth cluster

spec:
  identityProviders:
  - name: keystone
    mappingMethod: claim
    type: OpenID
    openID:
      clientID: <keystone-client-id>
      clientSecret:
        name: <secret-name>
      claims:
        preferredUsername:
        - preferred_username
        name:
        - name
        email:
        - email
      issuer: https://<keystone-url>/v3

oc create secret generic <secret-name> --from-literal=clientSecret=<keystone-client-secret> -n openshift-config
```
## In Keystone "OpenStack"
``` bash
openstack user create --domain default --password <password> <username>
openstack role add --user <username> --project <project-name> <role-name>

```
## To Verify in Cluster-OCP
``` bash
oc login --server=<ocp-api-url> --username=<keystone-username> --password=<keystone-password>
oc adm policy add-role-to-user <role> <username> -n <namespace>
```
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

## To Show All Api-Resources 
``` bash
oc api-resources
```

## To Show Cluster-Operator of your Cluster
``` bash
oc get co
```

## To make remote/ssh to master node
``` bash
ssh core@<Master-ip>
```
## To Export kubeconfig 
## Bastion-Host
``` bash
export KUBECONFIG=/etc/kubernetes/static-pod-resources/kube-apiserver-certs/secrets/node-kubeconfigs/lb-int.kubeconfig
```
## Master node
``` bash
ssh core@<master-ip>
export KUBECONFIG=/etc/kubernetes/static-pod-resources/kube-apiserver-certs/secrets/node-kubeconfigs/lb-int.kubeconfig
oc get nodes
```

## Solution of After restart OpenShift Cluter or network lost of machines , we should check the below 
1- go to contole planes as sequence  using ssh and check the below 
``` bash
systemctl status kubelet
systemctl restart kubelet
systemctl status crio
systemctl restart  crio
OR "Script"
for host in $HOSTS;do ssh core@$host sudo systemctl restart kubelet; ssh core@$host sudo systemctl restart crio; done

sudo -i
crictl ps "Show Running Containners"
```

## Check OCP Certificate Duration
``` bash
echo | openssl s_client -connect api-int.ocp-installation-test.lab.local:6443 | openssl x509 -noout -text
```

## To Upgrade Cluster-Version of OCP-Cluster
## Note : etcdbackup schedule task
https://access.redhat.com/labsinfo/ocpupgradegraph    "stable channel"

## Upgrade Cluster as CLI
``` bash
oc adm upgrade --to=<version>
```
## Delete stuck ns as force "terminiating status"
``` bash
NS=`oc get ns |grep Terminating | awk 'NR==1 {print $1}'` && oc get namespace <namespace-name> -o json   | tr -d "\n" | sed "s/\"finalizers\": \[[^]]\+\]/\"finalizers\": []/"   | oc replace --raw /api/v1/namespaces/<namespace-name>/finalize -f -
```
## Create NFS
``` bash
oc create namespace openshift-nfs-storage
oc process -f  https://raw.githubusercontent.com/openshift-examples/external-storage-nfs-client/main/openshift-template-nfs-client-provisioner.yaml  -p NFS_SERVER=172.16.226.90  -p NFS_PATH=/root/amro_dir/ocp_share  | oc apply -f -
```

## Image Registry on Nutanix Volume
https://opendocs.nutanix.com/openshift/post-install/

## Machine Config Operator ISSUE
https://access.redhat.com/solutions/4970731

``` bash
oc debug node/$node_name -- touch /host/run/machine-config-daemon-force
```
## Change ssh-keygen that installed in OCP-Cluster
https://access.redhat.com/solutions/3868301
1- For Worker-Nodes
2- For Master-Nodes

## Create ldap-cer integeration
``` bash
vim ldap-ca.yml

kind: ConfigMap
apiVersion: v1
metadata:
  name: ldap-ca
  namespace: openshift-config
data:
  ca.crt: |-
  <>
    -----END CERTIFICATE-----

oc apply -f ldap-ca.yml
oc edit oauths.config.openshift.io

apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    include.release.openshift.io/single-node-developer: "true"
    release.openshift.io/create-only: "true"
  creationTimestamp: "2023-03-16T10:31:29Z"
  generation: 5
  name: cluster
  ownerReferences:
  - apiVersion: config.openshift.io/v1
    kind: ClusterVersion
    name: version
    uid: ca844b4a-8be3-4324-936d-c061cca5d762
  resourceVersion: "151859623"
  uid: <uid-ocp-cluster>
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: <htpasswd-secret-name>
    mappingMethod: claim
    name: htpasswd
    type: HTPasswd
  - ldap:
      attributes:
        email: []
        id:
        - dn
        name:
        - cn
        preferredUsername:
        - name
      bindDN: <>
      bindPassword:
        name: <ldap-bind-password-secret>
      ca:
        name: ldap-ca
      insecure: false
      url: ldaps://HQDC01.hqdomain.com:636/dc=hqdomain,dc=com?userPrincipalName?sub?(&(objectclass=*)(|(memberOf:1.2.840.113556.1.4.1941:=CN=<>)))
    mappingMethod: claim
    name: LDAP
    type: LDAP

## Sync Manifestes-Gitlab-ldap with Ansible platform
## you have yaml files in Gitlab that defined yaml of OCP and i want to run this yaml file from Ansible-platform-Automation
1- Login in Gitlab and choose Repo that has yaml file

2- Login to Cluster as Cli
``` bash
oc login -u <username> -p <passsword>
oc whoami --show-server
https://<>:6443
oc whoami --show-console

```
![login cluster as cli](https://github.com/user-attachments/assets/9a7150b6-86c9-403b-82c8-00b42222083f)



3- Login to Cluster as Console > Secrets > ldap-bind-password-<id>
![console-1](https://github.com/user-attachments/assets/92d7dcde-7971-43a4-9a13-217eb3d833cd)

4- To know <ladp-password-<id>> Go to :
![Auth-1](https://github.com/user-attachments/assets/5d4aa3e7-59b1-4221-ac71-c36b29785463)  >> yaml >> find in yaml ldap

5- Login to Ansible 
platform Automation > Choose Projects > Projectname > Make Sync to Update
![Sync Project in Ansible](https://github.com/user-attachments/assets/298e4d23-d615-4eef-b1e7-576fd81ca2d5)


6- Check Creedentials of templete and select plabook.yaml that existing in gitlap-repo
![templete cred](https://github.com/user-attachments/assets/df320feb-4505-4ad2-b44b-0d34d21578e1)
7- Gitlab URL In OpenShitf-Porject
![Gitlab-url in ansible](https://github.com/user-attachments/assets/83f4714e-3c32-4671-b045-cbbf0fa99067)



## Show VIP of  [ Master-Node, Worker-Node and DMZ-Node ]
1- VIP Of Master-node
``` bash
oc whoami --show-server
ping -c 1 <api......>          Result is ip "VIP"
```
2- VIP Of Worker-Node
``` bash
oc whoami --show-console
ping -c 1 <console-openshift-......>
```
3- VIP of DMZ
``` bash
oc get ingresscontrollers.operator.openshift.io -A
oc edit ingresscontrollers.operator.openshift.io dmz-ingress-controller -n openshift-ingress-operator           "search line domain and make ping to the value of domain"
ping -c 1 <value-of-domain>
```
![VIP OF nodes](https://github.com/user-attachments/assets/ef2f106d-535b-4d0b-8c06-2f9cdbf30af3)
