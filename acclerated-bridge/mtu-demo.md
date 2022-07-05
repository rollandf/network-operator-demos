# MTU Demo

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

mstconfig -d d8:00.0 set SRIOV_EN=1
mstconfig -d d8:00.0 set NUM_OF_VFS=4 
```

## After reboot check state
```bash
mstconfig -e -d d8:00.0 q | grep -E "SRIOV_EN|NUM_OF_VFS"
```

## Create VF - not bind to driver

```bash
echo 0 > /sys/class/net/ens3f0np0/device/sriov_drivers_autoprobe

echo 4 > /sys/class/net/ens3f0np0/device/sriov_numvfs

echo 1 > /sys/class/net/ens3f0np0/device/sriov_drivers_autoprobe
```

Check with lspci

## switchdev

Check eswitch mode:

```bash
devlink dev eswitch show pci/0000:d8:00.0
pci/0000:d8:00.0: mode legacy inline-mode none encap-mode basic
```

Configure switchdev:
```bash
devlink dev eswitch set pci/0000:d8:00.0 mode switchdev
```

Representors should be created (check with ip link)


## Bind VFs to driver

```bash
VFS_PCI=($(lspci | grep "Mellanox" | grep "Virtual" | cut -d " " -f 1));
     for i in ${VFS_PCI[@]};
     do
         echo "binding VF $i";
         echo "0000:${i}" > /sys/bus/pci/drivers_probe;    
done
```

Verify:

```
lspci -nnk -d 15b3:101e
```

## Configure regular bridge

```bash
ip link add name accelbr type bridge
ip link set ens3f0np0 master accelbr
ip link set accelbr up
```

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

Restart Device Plugin if needed.

```
kubectl get nodes cloud-dev-11 -o json | jq .status.allocatable
```


## Acclerated bridge CNI

DS will deploy CNI binary to /opt/cni/bin/

Change image to lastest

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

Add `mtu` to config, for example 2000:

```
"mtu": 2000,
```


## Deploy workload pods

Apply [pod.yaml](./resources/pod.yaml) and [pod2.yaml](./resources/pod2.yaml)

`ping` between the pods on the secondary network IPs to verify connectivity.

```
ping 192.168.3.225 -I net1 -c 10 -M do -s 1972
```

`-I net1` Use specific interface
`-M do`   Prohibit fragmentation
`-c 10`   Count of pings
`-s 1972` Size of frame

`1972`:  2000 MTU -28 minus 20 bytes IPV4 header and 8 bytes for the ICMP header

## Check configuration on host

Check on host that VF and Representor have MTU configured:

```
ip link | grep 2000
```

Delete pods and check that MTU is restored.

==============================

# Reset configuration

```
ip link del name accelbr
devlink dev eswitch set pci/0000:d8:00.0 mode legacy
echo 0 > /sys/class/net/ens3f0np0/device/sriov_numvfs
```

==============================
# VFIO

## Enable IOMMU and IOMMU pass-through

IOMMU: Input-Output Memory Management Unit

Needed once only.

In `/etc/default/grub` add:

`intel_iommu=on iommu=pt` at the end of `GRUB_CMDLINE_LINUX`

and run: `grub2-mkconfig -o /boot/grub2/grub.cfg` and reboot.

## Create VFs unbound:
```
echo 0 > /sys/class/net/ens3f0np0/device/sriov_drivers_autoprobe

echo 4 > /sys/class/net/ens3f0np0/device/sriov_numvfs

echo 1 > /sys/class/net/ens3f0np0/device/sriov_drivers_autoprobe
```

## Set switchdev:
```
devlink dev eswitch set pci/0000:d8:00.0 mode switchdev
```

## Load VFIO driver:
```
lsmod  | grep vfio_pci
modprobe vfio-pci
lsmod  | grep vfio_pci
```

## Bind VFs to VFIO

Get script:
```
wget https://raw.githubusercontent.com/andre-richter/vfio-pci-bind/master/vfio-pci-bind.sh
chmod +x vfio-pci-bind.sh
```

Bind the VFs:

```bash
VFS_PCI=($(lspci | grep "Mellanox" | grep "Virtual" | cut -d " " -f 1));
     for i in ${VFS_PCI[@]};
     do
         echo "binding VF $i";
         ./vfio-pci-bind.sh "0000:${i}"
done
```

Verify:

```
lspci -nnk -d 15b3:101e
```

```
d8:00.2 Ethernet controller [0200]: Mellanox Technologies ConnectX Family mlx5Gen Virtual Function [15b3:101e]
	Subsystem: Mellanox Technologies Device [15b3:0083]
	Kernel driver in use: vfio-pci
	Kernel modules: mlx5_core
d8:00.3 Ethernet controller [0200]: Mellanox Technologies ConnectX Family mlx5Gen Virtual Function [15b3:101e]
	Subsystem: Mellanox Technologies Device [15b3:0083]
	Kernel driver in use: vfio-pci
	Kernel modules: mlx5_core
d8:00.4 Ethernet controller [0200]: Mellanox Technologies ConnectX Family mlx5Gen Virtual Function [15b3:101e]
	Subsystem: Mellanox Technologies Device [15b3:0083]
	Kernel driver in use: vfio-pci
	Kernel modules: mlx5_core
d8:00.5 Ethernet controller [0200]: Mellanox Technologies ConnectX Family mlx5Gen Virtual Function [15b3:101e]
	Subsystem: Mellanox Technologies Device [15b3:0083]
	Kernel driver in use: vfio-pci
	Kernel modules: mlx5_core

```

## Configure regular bridge

```bash
ip link add name accelbr type bridge
ip link set ens3f0np0 master accelbr
ip link set accelbr up
```


## KubeVirt

```
export RELEASE=v0.51.0

kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml

kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml

kubectl -n kubevirt wait kv kubevirt --for condition=Available

export VERSION=v0.41.0
wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
chmod +x virtctl-v0.41.0-linux-amd64
mv virtctl-v0.41.0-linux-amd64 /usr/local/bin/virtctl
```

Create [VM1](./resources/vm.yaml) and [VM2](./resources/vm2.yaml).

```
kubectl apply -f vm.yaml
kubectl apply -f vm2.yaml
kubectl get vms
kubectl get vmi
virtctl start testvm
virtctl start testvm2
kubectl get vms
kubectl get vmi
```

Check on host that Representor have MTU configured:
```
ip link | grep 2000
```

Exec into kubevirt pod to check env:

```
# env | grep SRIOVSWITCHDEV
PCIDEVICE_NVIDIA_COM_SRIOVSWITCHDEV=0000:d8:00.3
```

```
ssh admin@10.101.101.10
ip addr
sudo lshw -c network -businfo
```

`ping` between the pods on the secondary network IPs to verify connectivity.

```ping 192.168.3.225 -I net1 -c 10 -M do -s 1952```

`-I net1` Use specific interface
`-M do`   Prohibit fragmentation
`-c 10`   Count of pings
`-s 1952` Size of frame

`1952`:  2000 MTU -28 minus 40 bytes IPV6 header and 8 bytes for the ICMP header


Delete VMs and check that MTU is restored.