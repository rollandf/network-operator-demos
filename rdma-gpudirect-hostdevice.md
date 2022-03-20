## RDMA + GPU Direct + Host Device configuration

### GPU Operator:

Make sure nodes with GPUS are labeled with `feature.node.kubernetes.io/pci-10de.present`.

Configure GPU cluster Policy as [here](./resources/gpu-cluster-policy.yaml).

Note the driver spec:

```yaml
    rdma:
      enabled: true
      useHostMofed: false
```

Check that CPU is available in node resources:

```bash
oc get nodes co-node-22 -o json | jq .status.allocatable
{
  "cpu": "31500m",
  "ephemeral-storage": "179551099402",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "97600980Ki",
  "nvidia.com/gpu": "1",
  "nvidia.com/hostdev": "1",
  "pods": "250"
}
```

### Host Device configuration

See [here](./host-device.md)

### Workload pods

- Specify nodes
- Request 'hostdev' resource

NOTE: Make sure that the NICs are connected to same switch

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rdma-gpu-test-pod-1
  annotations:
    k8s.v1.cni.cncf.io/networks: example-hostdevice-network # name from HostDeviceNetwork
spec:
  nodeSelector:
    # Note: Replace hostname
    kubernetes.io/hostname: worker01
  restartPolicy: OnFailure
  containers:
  - image: harbor.mellanox.com/cloud-orchestration-dev/cuda-perftest-image:latest
    name: rdma-gpu-test-ctr
    securityContext:
      capabilities:
        add: [ "IPC_LOCK" ]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/hostdev: '1'
      requests:
        nvidia.com/gpu: 1
        nvidia.com/hostdev: '1'
```

Run two pods on different nodes.

Connect to shell:

```bash
oc rsh test-hostdev-pod
```

Lookup device name:

```bash
ls /sys/class/infiniband
mlx5_0
```

One first pod run server:

```bash
#server
ib_write_bw -d <device: opensource5_0 for example> -F --report_gbits -R -q 2 --use_cuda=0
```

```bash
#server example
ib_write_bw -d mlx5_0 -F -R --report_gbits --use_cuda=0

************************************
* Waiting for client to connect... *
************************************


initializing CUDA
Listing all CUDA devices in system:
CUDA device 0: PCIe address is 82:00

Picking device No. 0
[pid = 68, dev = 0] device name = [Tesla T4]
creating CUDA Ctx
making it the current CUDA Ctx
cuMemAlloc() of a 131072 bytes GPU buffer
allocated GPU buffer address at 00007f31d7000000 pointer=0x7f31d7000000
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_0
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
 local address: LID 0000 QPN 0x00a8 PSN 0x4a6afe
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:227
 remote address: LID 0000 QPN 0x00a8 PSN 0xb8dfd2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:232
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             10.46              10.42  		   0.019882
---------------------------------------------------------------------------------------
deallocating RX GPU buffer 00007f31d7000000
destroying current CUDA Ctx

```

On other pod run client:

```bash
ib_write_bw -d <device: mlx5_0 for example> -F -R --report_gbits --use_cuda=0  <ip of server>
```

```bash
#client example

ib_write_bw -d mlx5_0 -F -R --report_gbits --use_cuda=0 192.168.3.227

initializing CUDA
Listing all CUDA devices in system:
CUDA device 0: PCIe address is 82:00

Picking device No. 0
[pid = 17, dev = 0] device name = [Tesla T4]
creating CUDA Ctx
making it the current CUDA Ctx
cuMemAlloc() of a 131072 bytes GPU buffer
allocated GPU buffer address at 00007fcb25000000 pointer=0x7fcb25000000
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_0
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
 local address: LID 0000 QPN 0x00a8 PSN 0xb8dfd2
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:232
 remote address: LID 0000 QPN 0x00a8 PSN 0x4a6afe
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:227
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             10.46              10.42  		   0.019882
---------------------------------------------------------------------------------------
deallocating RX GPU buffer 00007fcb25000000
destroying current CUDA Ctx

```
