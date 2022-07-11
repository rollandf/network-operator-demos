# Accelerated Bridge CNI

https://wikinox.mellanox.com/pages/viewpage.action?spaceKey=SW&title=Accelerated+Linux+Bridge+Demo+manuals

## Nodes configuration

1. Install Centos on nodes
2. Upgrade to latest kernel

```bash
mv /etc/yum.repos.d/mlnx.repo ~
yum update -y
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
sudo dnf --enablerepo=elrepo-kernel install kernel-ml kernel-ml-devel
```

```bash
uname -a
Linux cloud-dev-11 5.17.6-1.el8.elrepo.x86_64 #1 SMP PREEMPT Fri May 6 09:03:21 EDT 2022 x86_64 x86_64 x86_64 GNU/Linux
```

**Note**:
In some servers, the following errors appears when configuring SR-IOV:
```bash
May 11 12:47:18 cloud-dev-11 kernel: mlx5_core 0000:d8:00.0: not enough MMIO resources for SR-IOV
```
```bash
May 11 12:47:18 cloud-dev-11 switchdev-configuration-before-nm.sh[1866]: /usr/local/bin/switchdev-configuration-before-nm.sh: line 37: echo: write error: Cannot allocate memory
```
This can be solved by adding the following kernel parameters:
```bash
grubby --update-kernel=ALL --args="pci=realloc"
```

3. Install OpenVswitch

Currently, SR-IOV Network Operator requires open vswitch to configure SR-IOV with Switchdev.

Install open vswitch on Centos 8 https://computingforgeeks.com/how-to-install-open-vswitch-on-centos-rhel/

Update with 8-stream in URL (or just move repo out):
```bash
yum install -y epel-release centos-release-openstack-train
mv /etc/yum.repos.d/CentOS-Messaging-rabbitmq.repo ~
mv /etc/yum.repos.d/ceph-nautilus.repo ~
vi /etc/yum.repos.d/CentOS-OpenStack-train.repo #Change: baseurl=http://mirror.centos.org/$contentdir/8-stream/cloud/$basearch/openstack-train/

sudo yum install openvswitch libibverbs
sudo systemctl enable --now openvswitch
```

4. Install kubernetes

## Install Network Operator + SR-IOV Network Operator:

```bash
helm install -n network-operator --kubeconfig=k --create-namespace --wait network-operator mellanox/network-operator --set operator.repository=harbor.mellanox.com/cloud-orchestration-dev --set sriovNetworkOperator.enabled=true --set sriov-network-operator.images.operator=harbor.mellanox.com/cloud-orchestration-dev/sriov-network-operator:network-operator-1.2.0-rc.1 --set sriov-network-operator.images.sriovConfigDaemon=harbor.mellanox.com/cloud-orchestration-dev/sriov-network-operator-config-daemon:network-operator-1.2.0-rc.1 --set ofedDriver.repository=harbor.mellanox.com/sw-linux-devops --set ofedDriver.version=5.6-1.0.3.3
```

## Deploy accelerated-bridge CNI via DaemonSet

Apply [bridge-cni-ds.yaml](./resources/bridge-cni-ds.yaml).

## [Optional] Compile and install accelerated-bridge CNI binary manually
If needed, compile and deploy manually the accelerated-bridge CNI binary.
```bash
git clone git@github.com:rollandf/accelerated-bridge-cni.git
cd accelerated-bridge-cni/
make
scp ./build/accelerated-bridge root@cloud-dev-66:/opt/cni/bin/
```

## Deploy NIC Cluster Policy

Apply [nic-cluster-policy.yaml](./resources/nic-cluster-policy.yaml).

## Deploy SR-IOV Network Policy

Apply [sriov-network-policy.yaml](./resources/sriov-network-policy.yaml).

## Configure bridge

### Using nmcli (recommended)
Using nmcli will make the configuration persistent after reboot.

```bash
nmcli con add type bridge ifname accelbr con-name accelbr bridge.stp no ipv4.method disabled ipv6.method link-local
nmcli con add type bridge-slave ifname ens3f0np0 master accelbr con-name ens3f0np0
nmcli con up ens3f0np0
nmcli con up accelbr
```

### Using IP link (not recommended)
Note that this configuration is not persisted after reboot.

```bash
ip link add name accelbr type bridge
ip link set ens3f0np0 master accelbr
ip -d link show
ip link set accelbr up
```

## Deploy Network Attachment

Apply [network-attachment.yaml](./resources/network-attachment.yaml).

## Deploy workload pods

Apply [pod.yaml](./resources/pod.yaml) and [pod2.yaml](./resources/pod2.yaml)

`ping` between the pods on the secondary network IPs to verify connectivity.

## Check that traffic is offloaded

```bash
# bridge fdb | grep offl
b8:ce:f6:8d:c8:75 dev ens3f0np0 extern_learn offload master accelbr 
b8:ce:f6:8d:c8:c5 dev ens3f0np0 extern_learn offload master accelbr 
b8:ce:f6:09:f8:8d dev ens3f0np0 extern_learn offload master accelbr 
b8:ce:f6:8d:c8:c4 dev ens3f0np0 extern_learn offload master accelbr
```

```bash
echo > /sys/kernel/debug/tracing/trace
echo mlx5:mlx5_esw_bridge_fdb_entry_init >> /sys/kernel/debug/tracing/set_event
echo mlx5:mlx5_esw_bridge_fdb_entry_cleanup >> /sys/kernel/debug/tracing/set_event
echo mlx5:mlx5_esw_bridge_fdb_entry_refresh >> /sys/kernel/debug/tracing/set_event
echo mlx5:mlx5_esw_bridge_vlan_create >> /sys/kernel/debug/tracing/set_event
echo mlx5:mlx5_esw_bridge_vlan_cleanup >> /sys/kernel/debug/tracing/set_event
echo mlx5:mlx5_esw_bridge_vport_init >> /sys/kernel/debug/tracing/set_event
echo mlx5:mlx5_esw_bridge_vport_cleanup >> /sys/kernel/debug/tracing/set_event
watch -n1 "cat /sys/kernel/debug/tracing/trace | tail -n 20"
```

```bash
# ip link show | grep master
8: ens3f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
26: ens3f1np1_1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
28: ens3f1np1_3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
```


## VF LAG


### Using nmcli (recommended)
Using nmcli will make the configuration persistent after reboot.

```bash
nmcli con add type bridge ifname accelbr con-name accelbr bridge.stp no ipv4.method disabled ipv6.method link-local
nmcli con add type bond con-name bond0 ifname bond0 mode 802.3ad
nmcli con add type bond-slave ifname ens3f0np0 master bond0 con-name ens3f0np0
nmcli con add type bond-slave ifname ens3f1np1 master bond0 con-name ens3f1np1
nmcli con modify bond0 master accelbr slave-type bridge
nmcli con up ens3f1np1
nmcli con up ens3f0np0
nmcli con up bond0
nmcli con up accelbr
```

### Using network-scripts files (not recommended)

Configure the bond + bridge in network-scripts to be loaded by Network Manager.

```
# cat << EOF >  /etc/sysconfig/network-scripts/ifcfg-accelbr-1
STP=no
BRIDGING_OPTS=group_address=01:80:C2:00:00:00
TYPE=Bridge
HWADDR=
MACADDR=04:3F:72:B2:C0:88
MTU=1500
ACCEPT_ALL_MAC_ADDRESSES=no
PROXY_METHOD=none
BROWSER_ONLY=no
ETHTOOL_OPTS="-K accelbr gro on gso on highdma on rx-gro-list off rx-udp-gro-forwarding off tso on txvlan on tx-checksum-ip-generic on tx-esp-segmentation on tx-fcoe-segmentation off tx-gre-csum-segmentation on tx-gre-segmentation on tx-gso-list off tx-gso-partial on tx-gso-robust off tx-ipxip4-segmentation on tx-ipxip6-segmentation on tx-nocache-copy off tx-scatter-gather-fraglist off tx-sctp-segmentation off tx-tcp6-segmentation on tx-tcp-ecn-segmentation on tx-tcp-mangleid-segmentation on tx-tunnel-remcsum-segmentation on tx-udp-segmentation on tx-udp_tnl-csum-segmentation on tx-udp_tnl-segmentation on tx-vlan-stag-hw-insert on"
IPV6INIT=yes
IPV6_AUTOCONF=no
DHCPV6_DUID=ll
DHCPV6_IAID=mac
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=accelbr
UUID=9eb9ebf4-3147-4a13-aaa4-f1e21e540584
DEVICE=accelbr
ONBOOT=yes
AUTOCONNECT_SLAVES=yes
LLDP=no
EOF
```

```
# cat << EOF >  /etc/sysconfig/network-scripts/ifcfg-bond0-1 
BONDING_OPTS="mode=802.3ad lacp_rate=fast miimon=100"
TYPE=Bond
BONDING_MASTER=yes
NAME=bond0
UUID=6e7c06e6-b315-44df-9d82-5b554134d60d
DEVICE=bond0
ONBOOT=yes
BRIDGE=accelbr
AUTOCONNECT_SLAVES=yes
LLDP=no
BRIDGE_UUID=9eb9ebf4-3147-4a13-aaa4-f1e21e540584
EOF
```

```
# cat << EOF >  /etc/sysconfig/network-scripts/ifcfg-ens3f0np0
DEVICE=ens3f0np0
NAME=bond0-slave
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=yes
EOF
```

```
# cat << EOF >  /etc/sysconfig/network-scripts/ifcfg-ens3f1np1 
DEVICE=ens3f1np1
NAME=bond0-slave
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
NM_CONTROLLED=yes
EOF
```


After running workload pods:
```
#  ip link | grep acc
25: ens3f1np1_0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
28: ens3f1np1_3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
31: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue master accelbr state UP mode DEFAULT group default qlen 1000
32: accelbr: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
```

## Bridge only - via network scripts (not recommended)

```
# cat /etc/sysconfig/network-scripts/ifcfg-accelbr 
STP=no
BRIDGING_OPTS=priority=32768
TYPE=Bridge
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=no
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
AUTOCONNECT_SLAVES=yes
NAME=accelbr
UUID=260d2ee8-72df-44c6-af75-64fc0cba9738
DEVICE=accelbr
ONBOOT=yes
```

```
# cat /etc/sysconfig/network-scripts/ifcfg-bridge-slave-ens3f0np0 
TYPE=Ethernet
NAME=bridge-slave-ens3f0np0
UUID=52089f1b-f66c-4d98-82a4-460e77470cc6
DEVICE=ens3f0np0
ONBOOT=yes
BRIDGE=accelbr
```

```
# ip link | grep accelbr
8: ens3f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
28: ens3f1np1_3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
29: ens3f1np1_4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master accelbr state UP mode DEFAULT group default qlen 1000
30: accelbr: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
```

### k8s NMSTATE

Currently, configuring using k8s-nmstate for configuring the bridgr is problematic.
Nmstate configure SRIOV_TOTAL_VFS even if it is not specified in required nmstate spec.
This cause reset of the settings in the NICS, removing the switchdev configuration.

NMSTATE examples:

[Bridge only](./resources/nmstate_bridge.yaml)
[Bridge + VF LAG](./resources/nmstate_vflag.yaml)
[Delete Bridge](./resources/delete_bridge.yaml)