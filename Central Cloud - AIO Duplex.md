# Host Setup

```
#1 Update OS
apt-get update


#2 Clone StarlingX tools repository
mkdir starlingx4
cd starlingx4
git clone https://opendev.org/starlingx/tools.git

#3 Install required packages
cd tools/deployment/libvirt/
bash install_packages.sh
apt install -y apparmor-profiles
apt-get install -y ufw
ufw disable
ufw status

#4 Download starlingx4 ISO
wget http://mirror.starlingx.cengn.ca/mirror/starlingx/release/4.0.1/centos/flock/outputs/iso/bootimage.iso
```

# Prepare virtual environment and server
```
#1 Create virtual server
bash setup_configuration.sh -c duplex -i ./bootimage.iso

This will create the XML virtual server definition for:
duplex-controller-0
duplex-controller-1

Install StarlingX Kubernetes on Virtual AIO-DX
#1 Start vm
virsh start cc-duplex-controller-0

#2 Install software on controller-0
virsh console cc-duplex-controller-0

1. First menu: Select ‘All-in-one Controller Configuration’
2. Second menu: Select ‘Serial Console’

wait 5-10 minutes.

#3 Login to controller-0
Log in using the username / password of “sysadmin” / “sysadmin”

#4 Set External connectivity
export CONTROLLER0_OAM_CIDR=10.10.20.3/24
export DEFAULT_OAM_GATEWAY=10.10.20.1
sudo ip address add $CONTROLLER0_OAM_CIDR dev enp7s1
sudo ip link set up dev enp7s1
sudo ip route add default via $DEFAULT_OAM_GATEWAY dev enp7s1

#5 Check configuration
ip a

#6 Create bootstrap template
cd ~
cat <<EOF > localhost.yml
system_mode: duplex
timezone: Asia/Jakarta
dns_servers:
  - 8.8.8.8
  - 8.8.4.4

external_oam_subnet: 10.10.20.0/24
external_oam_gateway_address: 10.10.20.1
external_oam_floating_address: 10.10.20.2
external_oam_node_0_address: 10.10.20.3
external_oam_node_1_address: 10.10.20.4

distributed_cloud_role: systemcontroller
management_subnet: 192.168.20.0/24
management_start_address: 192.168.20.2
management_end_address: 192.168.20.50

admin_username: admin
admin_password: <sysadmin password>
ansible_become_pass: <sysadmin password>
EOF

#7 Run the Ansible bootstrap playbook.
ansible-playbook /usr/share/ansible/stx-ansible/playbooks/bootstrap.yml

```

## Configure controller-0
```
#1 Load openrc
source /etc/platform/openrc

#2 Configure the OAM and MGMT interfaces of controller-0 and specify the attached networks.
OAM_IF=enp7s1
MGMT_IF=enp7s2
system host-if-modify controller-0 lo -c none
IFNET_UUIDS=$(system interface-network-list controller-0 | awk '{if ($6=="lo") print $4;}')
for UUID in $IFNET_UUIDS; do
    system interface-network-remove ${UUID}
done
system host-if-modify controller-0 $OAM_IF -c platform
system interface-network-assign controller-0 $OAM_IF oam
system host-if-modify controller-0 $MGMT_IF -c platform
system interface-network-assign controller-0 $MGMT_IF mgmt
system interface-network-assign controller-0 $MGMT_IF cluster-host

#3 Configure NTP Server
system ntp-modify ntpservers=0.id.pool.ntp.org,1.id.pool.ntp.org

#4 Configure ceph storage backend
system storage-backend-add ceph --confirmed

#5 Configure Data interface for controller-0.
DATA0IF=eth1000
DATA1IF=eth1001
export NODE=controller-0
PHYSNET0='physnet0'
PHYSNET1='physnet1'
SPL=/tmp/tmp-system-port-list
SPIL=/tmp/tmp-system-host-if-list
system host-port-list ${NODE} --nowrap > ${SPL}
system host-if-list -a ${NODE} --nowrap > ${SPIL}
DATA0PCIADDR=$(cat $SPL | grep $DATA0IF |awk '{print $8}')
DATA1PCIADDR=$(cat $SPL | grep $DATA1IF |awk '{print $8}')
DATA0PORTUUID=$(cat $SPL | grep ${DATA0PCIADDR} | awk '{print $2}')
DATA1PORTUUID=$(cat $SPL | grep ${DATA1PCIADDR} | awk '{print $2}')
DATA0PORTNAME=$(cat $SPL | grep ${DATA0PCIADDR} | awk '{print $4}')
DATA1PORTNAME=$(cat  $SPL | grep ${DATA1PCIADDR} | awk '{print $4}')
DATA0IFUUID=$(cat $SPIL | awk -v DATA0PORTNAME=$DATA0PORTNAME '($12 ~ DATA0PORTNAME) {print $2}')
DATA1IFUUID=$(cat $SPIL | awk -v DATA1PORTNAME=$DATA1PORTNAME '($12 ~ DATA1PORTNAME) {print $2}')

system datanetwork-add ${PHYSNET0} flat
system datanetwork-add ${PHYSNET1} flat

system host-if-modify -m 1500 -n data0 -c data ${NODE} ${DATA0IFUUID}
system host-if-modify -m 1500 -n data1 -c data ${NODE} ${DATA1IFUUID}
system interface-datanetwork-assign ${NODE} ${DATA0IFUUID} ${PHYSNET0}
system interface-datanetwork-assign ${NODE} ${DATA1IFUUID} ${PHYSNET1}

#6 Add an OSD on controller-0 for Ceph
system host-disk-list controller-0
system host-disk-list controller-0 | awk '/\/dev\/sdb/{print $2}' | xargs -i system host-stor-add controller-0 {}
system host-stor-list controller-0

####################################################
######### Openstack only configuration #############
####################################################

#7 Openstack role labeling
system host-label-assign controller-0 openstack-control-plane=enabled
system host-label-assign controller-0 openstack-compute-node=enabled
system host-label-assign controller-0 openvswitch=enabled

#8 Set up disk partition for nova-local volume group, which is needed for stx-openstack nova ephemeral disks.
export NODE=controller-0

export NODE=controller-0

echo ">>> Getting root disk info"
ROOT_DISK=$(system host-show ${NODE} | grep rootfs | awk '{print $4}')
ROOT_DISK_UUID=$(system host-disk-list ${NODE} --nowrap | grep ${ROOT_DISK} | awk '{print $2}')
echo "Root disk: $ROOT_DISK, UUID: $ROOT_DISK_UUID"

echo ">>>> Configuring nova-local"
NOVA_SIZE=34
NOVA_PARTITION=$(system host-disk-partition-add -t lvm_phys_vol ${NODE} ${ROOT_DISK_UUID} ${NOVA_SIZE})
NOVA_PARTITION_UUID=$(echo ${NOVA_PARTITION} | grep -ow "| uuid | [a-z0-9\-]* |" | awk '{print $4}')
system host-lvg-add ${NODE} nova-local
system host-pv-add ${NODE} nova-local ${NOVA_PARTITION_UUID}
sleep 2
```

## Unlock controller-0
```
#1 Unlock. It will reboot 10-15 minutes.
system host-unlock controller-0

#2 Login to controller-0 and check after reboot.
system host-list

#3 Show node configuration details.
system host-show controller-0
```

## Install software on controller-1
```
#1 Start cc-duplex-controller-1 VM on Host server.
virsh start cc-duplex-controller-1

#2 Enter and login to controller-1
virsh console duplex-controller-1

#3 From another terminal, login to controller-0 and check available node.
system host-list

#4 On controller-0, using the host id, set the personality of this host to ‘controller’
system host-update 2 personality=controller

#5 Check controller-1 console. It will take 10-15 minutes for bootstrapping.

#6 After bootstraping done, from controller-0 check available node again.
system host-list 
```

## Configure controller-1
```
#1 On controller-0 configure the OAM and MGMT interfaces of controller-1 and specify the attached networks. 
OAM_IF=enp7s1
system host-if-modify controller-1 $OAM_IF -c platform
system interface-network-assign controller-1 $OAM_IF oam
system interface-network-assign controller-1 mgmt0 cluster-host

####################################################
######### Openstack only configuration #############
####################################################

#2 Configure data interface for controller-1
DATA0IF=eth1000
DATA1IF=eth1001
export NODE=controller-1
PHYSNET0='physnet0'
PHYSNET1='physnet1'
SPL=/tmp/tmp-system-port-list
SPIL=/tmp/tmp-system-host-if-list
system host-port-list ${NODE} --nowrap > ${SPL}
system host-if-list -a ${NODE} --nowrap > ${SPIL}
DATA0PCIADDR=$(cat $SPL | grep $DATA0IF |awk '{print $8}')
DATA1PCIADDR=$(cat $SPL | grep $DATA1IF |awk '{print $8}')
DATA0PORTUUID=$(cat $SPL | grep ${DATA0PCIADDR} | awk '{print $2}')
DATA1PORTUUID=$(cat $SPL | grep ${DATA1PCIADDR} | awk '{print $2}')
DATA0PORTNAME=$(cat $SPL | grep ${DATA0PCIADDR} | awk '{print $4}')
DATA1PORTNAME=$(cat  $SPL | grep ${DATA1PCIADDR} | awk '{print $4}')
DATA0IFUUID=$(cat $SPIL | awk -v DATA0PORTNAME=$DATA0PORTNAME '($12 ~ DATA0PORTNAME) {print $2}')
DATA1IFUUID=$(cat $SPIL | awk -v DATA1PORTNAME=$DATA1PORTNAME '($12 ~ DATA1PORTNAME) {print $2}')

system datanetwork-add ${PHYSNET0} vlan
system datanetwork-add ${PHYSNET1} vlan

system host-if-modify -m 1500 -n data0 -c data ${NODE} ${DATA0IFUUID}
system host-if-modify -m 1500 -n data1 -c data ${NODE} ${DATA1IFUUID}
system interface-datanetwork-assign ${NODE} ${DATA0IFUUID} ${PHYSNET0}
system interface-datanetwork-assign ${NODE} ${DATA1IFUUID} ${PHYSNET1}

#3 Add an OSD on controller-1 for Ceph
echo ">>> Add OSDs to primary tier"
system host-disk-list controller-1
system host-disk-list controller-1 | awk '/\/dev\/sdb/{print $2}' | xargs -i system host-stor-add controller-1 {}
system host-stor-list controller-1

#4 Node roles labeling for openstack.
system host-label-assign controller-1 openstack-control-plane=enabled
system host-label-assign controller-1 openstack-compute-node=enabled
system host-label-assign controller-1 openvswitch=enabled

#5 Set up disk partition for nova-local volume group, which is needed for stx-openstack nova ephemeral disks.
export NODE=controller-1

echo ">>> Getting root disk info"
ROOT_DISK=$(system host-show ${NODE} | grep rootfs | awk '{print $4}')
ROOT_DISK_UUID=$(system host-disk-list ${NODE} --nowrap | grep ${ROOT_DISK} | awk '{print $2}')
echo "Root disk: $ROOT_DISK, UUID: $ROOT_DISK_UUID"

echo ">>>> Configuring nova-local"
NOVA_SIZE=34
NOVA_PARTITION=$(system host-disk-partition-add -t lvm_phys_vol ${NODE} ${ROOT_DISK_UUID} ${NOVA_SIZE})
NOVA_PARTITION_UUID=$(echo ${NOVA_PARTITION} | grep -ow "| uuid | [a-z0-9\-]* |" | awk '{print $4}')
system host-lvg-add ${NODE} nova-local
system host-pv-add ${NODE} nova-local ${NOVA_PARTITION_UUID}
```

## Unlock controller-1
```
#1 Unlock. Wait 10-15 minutes.
system host-unlock controller-1

#2 Check after finish.
system host-list

#3 Check node details.
system host-show controller-1

#4 If host status `degraded`, make sure data syncronization has been finished.
drbd-overview

#5 Check again
system host-list
```

## Deploy Kuberentes Dashboard
```
#1 Install Objek Kubernetes Dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml

#2 Edit service kubernetes-dashboard
kubectl edit -n kubernetes-dashboard svc kubernetes-dashboard
...
spec:
  clusterIP: 10.108.195.24
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30000
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
...

kubectl -n kubernetes-dashboard get pods
kubectl -n kubernetes-dashboard get svc

#3 Create Service Account and Cluster Role Binding
vim sa-admin-user.yaml
...
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
...

kubectl apply -f sa-admin-user.yaml

vim cr-binding-admin-user.yaml
...
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
...

kubectl apply -f cr-binding-admin-user.yaml
kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts

#4 Menampilkan service account token
kubectl get secrets -n kubernetes-dashboard
kubectl describe secret [secret-name]

Access Dashboard Kubernetes:
https://oam_floating_address:30000
```

## Deploy Openstack
```
#1 Download Chart
http://mirror.starlingx.cengn.ca/mirror/starlingx/release/4.0.1/centos/containers/outputs/helm-charts/stx-openstack-1.0-49-centos-stable-latest.tgz

#2 Upload chart
source /etc/platform/openrc
system application-upload stx-openstack-1.0-49-centos-stable-latest.tgz

#3 After chart uploaded, apply it.
system application-apply stx-openstack

#4 Wait for the activation of stx-openstack to complete.
watch -n 5 system application-list
```
