## RDMA + Host Device configuration


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
  name: test-hostdev-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: example-hostdevice-network # name from HostDeviceNetwork
spec:
  nodeSelector:
    # Note: Replace hostname
    kubernetes.io/hostname: worker01
  containers:
    - name: test-hostdev-pod
      image: mellanox/rping-test
      imagePullPolicy: IfNotPresent
      command: [ "/bin/bash", "-c", "--" ]
      args: [ "while true; do sleep 300; done;" ]
      securityContext:
        capabilities:
          add: [ "IPC_LOCK" ]
      resources:
        requests:
          nvidia.com/hostdev: '1'
        limits:
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
ib_write_bw -d <device: opensource5_0 for example> -F -R --report_gbits
```

```bash
#server example
ib_write_bw -d mlx5_0  -F -R --report_gbits

************************************
* Waiting for client to connect... *
************************************

---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_0
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
 local address: LID 0000 QPN 0x01ae PSN 0x39286b
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:225
 remote address: LID 0000 QPN 0x00aa PSN 0x8ef67f
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:226
---------------------------------------------------------------------------------------

```

On other pod run client:

```bash
ib_write_bw -d <device: mlx5_0 for example> -F -R --report_gbits <ip of server>
```

```bash
#client example
ib_write_bw -d  mlx5_0 -F -R --report_gbits  192.168.3.225

---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF		Device         : mlx5_0
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
 local address: LID 0000 QPN 0x00aa PSN 0x8ef67f
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:226
 remote address: LID 0000 QPN 0x01ae PSN 0x39286b
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:192:168:03:225
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[Gb/sec]    BW average[Gb/sec]   MsgRate[Mpps]
 65536      5000             92.50              92.50  		   0.176426
---------------------------------------------------------------------------------------

```
