### Whole NIC allocation/Host Device (Similar to PCU passthrough in VM):

- OFED installs NVIDIA drivers
- SRIOV device plugin detects Mellanox NICS that are RDMA capable and create resource on Node

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
  sriovDevicePlugin:
    image: sriov-device-plugin
    repository: docker.io/nfvpe
    version: v3.3
    config: |
      {
        "resourceList": [
            {
                "resourcePrefix": "nvidia.com",
                "resourceName": "hostdev",
                "selectors": {
                    "vendors": ["15b3"],
                    "isRdma": true
                }
            }
        ]
      }
```

Running pods output example with two workers:

```bash
oc get pods -n nvidia-network-operator-resources 
NAME                        READY   STATUS    RESTARTS   AGE
mofed-rhcos4.10-ds-9jnbs    1/1     Running   0          23m
mofed-rhcos4.10-ds-nwvmm    1/1     Running   0          23m
sriov-device-plugin-4vk98   1/1     Running   0          5m32s
sriov-device-plugin-8wqwm   1/1     Running   0          4m52s
```

Check if `hostdev` resource is available on node:

```bash
oc get nodes test-infra-cluster-3cfa70b8-worker-1 -o json | jq .status.allocatable
{
  "cpu": "1500m",
  "ephemeral-storage": "18833769646",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "7703760Ki",
  "nvidia.com/hostdev": "1",
  "pods": "250"
}

```

### HostDeviceNetwork

- Creates NetworkAttachmentDefinition for Multus secondary network


```yaml
apiVersion: mellanox.com/v1alpha1
kind: HostDeviceNetwork
metadata:
  name: example-hostdevice-network
spec:
  networkNamespace: "default" # pods namespace
  resourceName: "hostdev" # resource as created in NicClusterPolicy
  ipam: |
    {
      "type": "whereabouts",
      "range": "192.168.3.225/28",
      "exclude": [
       "192.168.3.229/30",
       "192.168.3.236/32"
      ]
    }
```

Verify that NetworkAttachmentDefinition is created:

```bash
oc get network-attachment-definitions
NAME                         AGE
example-hostdevice-network   103s
```
