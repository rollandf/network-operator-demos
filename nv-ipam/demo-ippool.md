# IP Pool CRD Demo

## IP Pool CRD

Design document is available [here](https://docs.google.com/document/d/15j6aNYVxqUbL_g4V7x0nK27rrRjNvbL_rqS9UZdh0M4/edit?usp=sharing).

IPPool CR [example](./resources/pool1.yaml).

## Upgrade flow

### Install Network Operator 23.7

Enable NV-IPAM v0.0.3 with config, Multus and CNI Plugins. [Values example](./resources/23-7-values.yaml)

```json
{
    "pools": {
        "pool1": { "subnet": "192.168.0.0/16", "perNodeBlockSize": 100 ,"gateway": "192.168.0.1"},
        "pool2": { "subnet": "172.16.0.0/16", "perNodeBlockSize": 50 , "gateway": "172.16.0.1"}
    },
    "nodeSelector": {"kubernetes.io/os": "linux"}
}

```

Helm install:

```
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install -n network-operator --create-namespace --version v23.7.0 -f ./resources/23-7-values.yaml network-operator nvidia/network-operator
```

Check Network Operators Pods:
```
kubectl -n network-operator get pods
```

Check NV IPAM IP allocations on Nodes:
```
kubectl get nodes -o=custom-columns='NAME:metadata.name,ANNOTATION:metadata.annotations.ipam\.nvidia\.com/ip-blocks'
```

### Install Network Operator 23.10 beta

Download beta Helm chart:
```
mkdir network-operator-helm-charts && cd network-operator-helm-charts

export NGC_API_KEY=<YOUR_NGC_TOKEN>
export NETWORK_OPERATOR_VERSION=23.10.0-beta.1

helm fetch https://helm.ngc.nvidia.com/nvstaging/mellanox/charts/network-operator-$NETWORK_OPERATOR_VERSION.tgz --username="\$oauthtoken" --password=$NGC_API_KEY

ls network-operator-*.tgz | xargs -n 1 tar xf
```

Create Secret for pulling Beta image:
```
kubectl -n network-operator create secret docker-registry ngc-image-secret --docker-server=nvcr.io --docker-username="\$oauthtoken" --docker-password=$NGC_API_KEY
```

Add Secret to `imagePullSecrets` in values file.

Enable NV-IPAM v0.1.0 without config, Multus and CNI Plugins. [Values example](./resources/23-10-values.yaml)

Run Helm upgrade:

```
helm upgrade -n network-operator  -f ../resources/23-10-values.yaml network-operator ./network-operator
```

Check Network Operators Pods:
```
kubectl -n network-operator get pods
```

### Verify upgrade

Check IPPools CR are created:

```
kubectl get ippools.nv-ipam.nvidia.com -n network-operator
```

Verify allocations:
```
kubectl get ippools.nv-ipam.nvidia.com -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"} {range .status.allocations[*]}{"\t"}{.nodeName} => Start IP: {.startIP} End IP: {.endIP}{"\n"}{end}{"\n"}{end}'
```

Verify Nodes annotations were cleared:
```
kubectl get nodes -o=custom-columns='NAME:metadata.name,ANNOTATION:metadata.annotations.ipam\.nvidia\.com/ip-blocks'
```

Verify that NV IPAM ConfigMap was deleted:
```
kubectl get cm -n network-operator nvidia-k8s-ipam-config
```

### Verify Pod IP allocation

Create NetworkAttachmentDefinition:

```
kubectl apply -f ./resources/nad.yaml
```

Create Pod:
```
kubectl apply -f ./resources/pod.yaml
```

Check Pod IP allocation:
```
kubectl describe pods test-pod-1 
```

Delete Pod
```
kubectl delete pod test-pod-1 
```

## Node Selector Per IP Pool

Edit Node Selector in IP Pool 1. (`color=blue`)
```
kubectl edit -n network-operator ippools.nv-ipam.nvidia.com pool1
```

Check IP Pool Status. No allocations on Pool1, as no Node match the selector.
```
kubectl get ippools.nv-ipam.nvidia.com -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"} {range .status.allocations[*]}{"\t"}{.nodeName} => Start IP: {.startIP} End IP: {.endIP}{"\n"}{end}{"\n"}{end}'
```

Label Host C with color label:
```
kubectl label node host-c color=blue
```

Check IP Pool Status. Host C should get an IP allocation on Pool1.
```
kubectl get ippools.nv-ipam.nvidia.com -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"} {range .status.allocations[*]}{"\t"}{.nodeName} => Start IP: {.startIP} End IP: {.endIP}{"\n"}{end}{"\n"}{end}'
```

## Event on failure to allocate IPs on Node

Edit Subnet in IP Pool 2. (`172.16.0.0/26`) This subnet allow 64 IPs.

```
kubectl edit -n network-operator ippools.nv-ipam.nvidia.com pool2
```

Check IP Pool Status. Only one allocation on Pool2.
```
kubectl get ippools.nv-ipam.nvidia.com -A -o jsonpath='{range .items[*]}{.metadata.name}{"\n"} {range .status.allocations[*]}{"\t"}{.nodeName} => Start IP: {.startIP} End IP: {.endIP}{"\n"}{end}{"\n"}{end}'
```

Check Events on IPPool 2.
```
kubectl describe -n network-operator ippools.nv-ipam.nvidia.com pool2
```

```
Events:
  Type     Reason        Age                  From              Message
  ----     ------        ----                 ----              -------
  Warning  NoFreeRanges  8m32s (x8 over 15m)  IPPoolController  failed to allocate IPs on Node: host-c
  Warning  NoFreeRanges  8m32s (x8 over 15m)  IPPoolController  failed to allocate IPs on Node: k8s-master
  Warning  NoFreeRanges  30s (x14 over 15m)   IPPoolController  failed to allocate IPs on Node: host-b
```

Or use:

```
kubectl get events -n network-operator
```