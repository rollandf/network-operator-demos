Based on:
https://confluence.nvidia.com/display/SW/Accelerated+Linux+Bridge+Demo+manuals

# Hosts

## Centos 8.5 with updated kernel

```bash
cat /etc/redhat-release
CentOS Linux release 8.5.2111

uname -a
Linux cloud-dev-11 5.17.6-1.el8.elrepo.x86_64 #1 SMP PREEMPT Fri May 6 09:03:21 EDT 2022 x86_64 x86_64 x86_64 GNU/Linux
```

**Note!!**
 cx6dx only

## Enable SRIOV and NUM of VF - One time only
```bash
lspci | grep llan

ll /sys/bus/pci/devices/0000\:d8\:00.0/net/
ll /sys/bus/pci/devices/0000\:d8\:00.1/net/

mstconfig -d d8:00.0 set SRIOV_EN=1
mstconfig -d d8:00.1 set SRIOV_EN=1

mstconfig -d d8:00.0 set NUM_OF_VFS=4 
mstconfig -d d8:00.1 set NUM_OF_VFS=4
```

## After reboot check state
```bash
mstconfig -e -d d8:00.0 q | grep -E "SRIOV_EN|NUM_OF_VFS"
mstconfig -e -d d8:00.1 q | grep -E "SRIOV_EN|NUM_OF_VFS"
```

## Create VF - not bind to driver

```bash
echo 0 > /sys/class/net/ens3f0np0/device/sriov_drivers_autoprobe
echo 0 > /sys/class/net/ens3f1np1/device/sriov_drivers_autoprobe

echo 4 > /sys/class/net/ens3f0np0/device/sriov_numvfs
echo 4 > /sys/class/net/ens3f1np1/device/sriov_numvfs    

echo 1 > /sys/class/net/ens3f0np0/device/sriov_drivers_autoprobe
echo 1 > /sys/class/net/ens3f1np1/device/sriov_drivers_autoprobe
```

Check with lspci

## switchdev

Check eswitch mode:

```bash
devlink dev eswitch show pci/0000:d8:00.1
pci/0000:d8:00.1: mode legacy inline-mode none encap-mode basic
```

Configure switchdev:
```bash
devlink dev eswitch set pci/0000:d8:00.1 mode switchdev
devlink dev eswitch set pci/0000:d8:00.0 mode switchdev
```

Representors should be created (check with ip link)

## Only first time configure bridge_bond+slaves (before bind)

Supported Bond modes are 1 active-backup ,2 balance-xor ,4 802.3ad

```bash
nmcli con add type bridge ifname accelbr con-name accelbr bridge.stp no ipv4.method disabled ipv6.method link-local
nmcli con add type bond con-name bond0 ifname bond0 mode 802.3ad
nmcli con modify bond0 master accelbr slave-type bridge

nmcli con add type bond-slave ifname ens3f0np0 master bond0 con-name ens3f0np0
nmcli con add type bond-slave ifname ens3f1np1 master bond0 con-name ens3f1np1

nmcli con up ens3f1np1
nmcli con up ens3f0np0
nmcli con up bond0
nmcli con up accelbr
nmcli con show
```

## Bind VFs to driver

```bash
modprobe vfio-pci
lsmod  | grep vfio_pci

VFS_PCI=($(lspci | grep "Mellanox" | grep "Virtual" | cut -d " " -f 1));
     for i in ${VFS_PCI[@]};
     do
         echo "binding VF $i";
         echo "0000:${i}" > /sys/bus/pci/drivers_probe;    
done
```

**Note!!**

Add PF to bond after VF bind will cause syndrome error:

```bash
nmcli con add type bond-slave ifname ens3f0np0 master bond0 con-name ens3f0np0
nmcli con add type bond-slave ifname ens3f1np1 master bond0 con-name ens3f1np1
```

From dmesg:

```bash
[  209.847235] mlx5_core 0000:d8:00.0: mlx5_cmd_check:790:(pid 888): CREATE_LAG(0x840) op_mod(0x0) failed, status bad parameter(0x3), syndrome (0x7d49cb)
[  209.847661] mlx5_core 0000:d8:00.0: mlx5_create_lag:281:(pid 888): Failed to create LAG (-22)
[  209.847835] mlx5_core 0000:d8:00.0: mlx5_activate_lag:338:(pid 888): Failed to activate VF LAG
               Make sure all VFs are unbound prior to VF LAG activation or deactivation
[  209.848390] mlx5_core 0000:d8:00.0: lag map port 1:1 port 2:1 shared_fdb:1 mode:queue_affinity
[  209.848452] mlx5_core 0000:d8:00.0: mlx5_cmd_check:790:(pid 888): CREATE_LAG(0x840) op_mod(0x0) failed, status bad parameter(0x3), syndrome (0x7d49cb)
[  209.848995] mlx5_core 0000:d8:00.0: mlx5_create_lag:281:(pid 888): Failed to create LAG (-22)
[  209.849270] mlx5_core 0000:d8:00.0: mlx5_activate_lag:338:(pid 888): Failed to activate VF LAG
               Make sure all VFs are unbound prior to VF LAG activation or deactivation

```

## Host reboot
After reboot Network Manager brings all interfaces up

The configuration steps are needed again, without the nmcli commands.
 - Create VFs without binding
 - Configure switchdev
 - Bind Vfs

# Kubernetes

## Multus

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick-plugin.yml
```


## SR-IOV device plugin

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sriovdp-config
  namespace: kube-system
data:
  config.json: |
    {
        "resourceList":  [
            {
                "resourcePrefix": "nvidia.com",
                "resourceName": "sriovswitchdev",
                "selectors": {
                    "vendors": ["15b3"],
                    "devices": ["101e"]
                }
            }
        ]
 
    }
```

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/sriov-network-device-plugin/master/deployments/k8s-v1.16/sriovdp-daemonset.yaml
```

kubectl get nodes cloud-dev-11 -o json | jq .status.allocatable


## Acclerated bridge CNI

DS will deploy CNI binary to /opt/cni/bin/

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/accelerated-bridge-cni/master/images/k8s-v1.16/accelerated-bridge-cni-daemonset.yaml
```


## IPAM

```bash
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/daemonset-install.yaml
kubectl apply -f  https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/ip-reconciler-job.yaml

kubectl apply -f  https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/whereabouts.cni.cncf.io_ippools.yaml

kubectl apply -f  https://raw.githubusercontent.com/k8snetworkplumbingwg/whereabouts/master/doc/crds/whereabouts.cni.cncf.io_overlappingrangeipreservations.yaml
```


## Network Attachment

Apply [network-attachment.yaml](./resources/network-attachment.yaml).


## Deploy workload pods

Apply [pod.yaml](./resources/pod.yaml) and [pod2.yaml](./resources/pod2.yaml)

`ping` between the pods on the secondary network IPs to verify connectivity.


## Check configuration on host

```bash
ip link  | grep accelbr

bridge fdb | grep offl
```

## Check traffic is offload

Run TCPDump on representor
```bash
tcpdump -leeni et12 icmp
```

Flush bridge DB
```bash
ip link set accelbr type bridge fdb_flush
```

We should see a couple of packets and then no traffic.