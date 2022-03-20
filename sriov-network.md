## SRIOV configuration


### NIC Cluster Policy

- OFED installs NVIDIA drivers


```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  ofedDriver:
    image: mofed
    repository: harbor.mellanox.com/sw-linux-devops
    version: 5.5-2.1.7.0-1
    startupProbe:
      initialDelaySeconds: 10
      periodSeconds: 20
    livenessProbe:
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      initialDelaySeconds: 10
      periodSeconds: 30
```

### SRIOV Operator Config:

A default `SriovOperatorConfig` is created by the SRIOV Netowrk Operator.

Update the [SriovOperatorConfig](./resources/sriov-network-operator-config.yaml).

Note:

```yaml
    resourceName: sriovlegacy
```

### SRIOV Node Policy:

Apply [SriovNetworkNodePolicy](./resources/sriov-network-node-policy-legacy.yaml)

Verify resources available on node:

```yaml
# oc get nodes co-node-22 -o json | jq .status.allocatable
{
  "cpu": "31500m",
  "ephemeral-storage": "179551099402",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "97600980Ki",
  "nvidia.com/gpu": "1",
  "nvidia.com/hostdev": "0",
  "openshift.io/sriovlegacy": "8",
  "pods": "250"
}
```

Note that the prefix of the resource is 'openshift.io'.


### SRIOV Network:

Apply [SriovNetwork](./resources/sriov-network.yaml)

### Workload pods

#### RDMA only

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-sriov-pod-0
  annotations:
    k8s.v1.cni.cncf.io/networks: example-sriov-network
spec:
  nodeSelector:
    # Note: Replace hostname
    kubernetes.io/hostname: worker01
    # or use:
    # feature.node.kubernetes.io/pci-15b3.sriov.capable: "true"
  containers:
    - name: test-sriov-pod
      image: mellanox/rping-test
      imagePullPolicy: IfNotPresent
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300; done;" ]
      securityContext:
        capabilities:
          add: [ "IPC_LOCK" ]
      resources:
        requests:
          openshift.io/sriovlegacy: "1"
        limits:
          openshift.io/sriovlegacy: "1"

```

Run two pods on different nodes.

Connect to shell:

```bash
oc rsh test-sriov-pod-0
```

Lookup device name:

```bash
ls /sys/class/infiniband
mlx5_0  mlx5_1  mlx5_10  mlx5_2  mlx5_3  mlx5_4  mlx5_5  mlx5_6  mlx5_7  mlx5_8  mlx5_9
```

There can be multiple devices. To find the right suffix:

```bash
ls /dev/infiniband/
issm8  rdma_cm  umad8  uverbs8
```

So the device here will be `mlx5_8`.

One first pod run server:

```bash
#server
ib_write_bw -d <device: opensource5_0 for example> -F -R --report_gbits
```

```bash
#server example
ib_write_bw -d  mlx5_4  -F -R --report_gbits

************************************
* Waiting for client to connect... *
************************************


---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_4
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 2
 Max inline data : 0[B]
 rdma_cm QPs	 : ON
 Data ex. method : rdma_cm
---------------------------------------------------------------------------------------
 Waiting for client rdma_cm QP to connect
 Please run the same command with the IB/RoCE interface IP
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x01c8 PSN 0x3b5954
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:225
 remote address: LID 0000 QPN 0x0208 PSN 0x4767f2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:226
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             92.56              92.56  		   0.176544
---------------------------------------------------------------------------------------
```


On other pod run client:

```bash
ib_write_bw -d <device: mlx5_0 for example> -F -R --report_gbits <ip of server>
```

```bash
#client example
ib_write_bw -d  mlx5_8 -F -R --report_gbits 192.168.4.225

---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_8
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 100
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 2
 Max inline data : 0[B]
 rdma_cm QPs	 : ON
 Data ex. method : rdma_cm
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0208 PSN 0x4767f2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:226
 remote address: LID 0000 QPN 0x01c8 PSN 0x3b5954
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:225
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             92.56              92.56  		   0.176544
--------------------------------------------------------------------------------------
```


### RDMA + GPU Direct

Similar as RDMA, requesting also GPU resource

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-gpu-test-pod-0
  annotations:
    k8s.v1.cni.cncf.io/networks: example-sriov-network
spec:
  nodeSelector:
    # Note: Replace hostname
    kubernetes.io/hostname: co-node-21
  restartPolicy: OnFailure
  containers:
  - image: harbor.mellanox.com/cloud-orchestration-dev/cuda-perftest:ubuntu20.04-cuda-devel-11.6.0-rdma-39-perftest-4.5
    name: rdma-gpu-test-ctr
    securityContext:
      capabilities:
        add: [ "IPC_LOCK" ]
    resources:
      limits:
        nvidia.com/gpu: 1
        openshift.io/sriovlegacy: "1"
      requests:
        nvidia.com/gpu: 1
        openshift.io/sriovlegacy: "1"
```

```yaml
# server example
ib_write_bw -d mlx5_5 -F -R --report_gbits --use_cuda=0

************************************
* Waiting for client to connect... *
************************************

initializing CUDA
Listing all CUDA devices in system:
CUDA device 0: PCIe address is 82:00

Picking device No. 0
[pid = 18, dev = 0] device name = [Tesla T4]
creating CUDA Ctx
making it the current CUDA Ctx
cuMemAlloc() of a 131072 bytes GPU buffer
allocated GPU buffer address at 00007f6eff000000 pointer=0x7f6eff000000
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_5
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: ON
 WARNING: CPU is not PCIe relaxed ordering compliant.
 WARNING: You should disable PCIe RO with `--disable_pcie_relaxed` for both server and client.
 ibv_wr* API     : ON
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs	 : ON
 Data ex. method : rdma_cm
---------------------------------------------------------------------------------------
 Waiting for client rdma_cm QP to connect
 Please run the same command with the IB/RoCE interface IP
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0088 PSN 0x6f0bef
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:225
 remote address: LID 0000 QPN 0x01ca PSN 0x6f0bef
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:226
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             10.68              10.50  		   0.020033
---------------------------------------------------------------------------------------
deallocating RX GPU buffer 00007f6eff000000
destroying current CUDA Ctx
```

```yaml
#client example

ib_write_bw -d mlx5_4 -F -R --report_gbits --use_cuda=0 192.168.4.225
initializing CUDA
Listing all CUDA devices in system:
CUDA device 0: PCIe address is 82:00

Picking device No. 0
[pid = 18, dev = 0] device name = [Tesla T4]
creating CUDA Ctx
making it the current CUDA Ctx
cuMemAlloc() of a 131072 bytes GPU buffer
allocated GPU buffer address at 00007f3813000000 pointer=0x7f3813000000
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_4
 Number of qps   : 1		Transport type : IB
 Connection type : RC		Using SRQ      : OFF
 PCIe relax order: ON
 WARNING: CPU is not PCIe relaxed ordering compliant.
 WARNING: You should disable PCIe RO with `--disable_pcie_relaxed` for both server and client.
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 3
 Max inline data : 0[B]
 rdma_cm QPs	 : ON
 Data ex. method : rdma_cm
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x01ca PSN 0x6f0bef
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:226
 remote address: LID 0000 QPN 0x0088 PSN 0x6f0bef
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:04:225
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             10.68              10.50  		   0.020033
---------------------------------------------------------------------------------------
deallocating RX GPU buffer 00007f3813000000
destroying current CUDA Ctx
```