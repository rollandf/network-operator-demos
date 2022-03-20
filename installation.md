# Network Operator installation for OpenShift

## 1. Cluster-wide Entitlement

Follow instructions in [GPU Operator docs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/cluster-entitlement.html).

I used the following subscription: `60 Day Product Trial of Red Hat OpenShift Container Platform, Self-Supported (32 Cores)`.

If needed, open an account in [developers.redhat.com](developers.redhat.com) for a 1 year subscription.

## 2. Install Node Feature Discovery

Create Namespace, Operator Group, Subscription from [here](./resources/nfd.yaml).

Create NodeFeatureDiscovery CR from [here](./resources/nfdcr.yaml).

Note that the configuration include also GPU configuration ("03").

Verify nodes labels:
- `feature.node.kubernetes.io/pci-15b3.present` for Networking
- `feature.node.kubernetes.io/pci-10de.present` for GPU

Source: [Red Hat docs](https://docs.openshift.com/container-platform/4.10/hardware_enablement/psap-node-feature-discovery-operator.html).


## 4. Create Network Operator namespace:

```bash
oc create ns nvidia-network-operator
```

Workaround for issue #[308](https://github.com/Mellanox/network-operator/issues/308)
```bash
oc create ns nvidia-network-operator-resources
```

## 5. NGC staging image registry access (If needed) - TODO check this work
oc create secret docker-registry ngc-secret \
  --docker-server=nvcr.io --docker-username=’$oauthtoken’ \
  --docker-password=<NGC API KEY> \
  --docker-email=<email-id> -n nvidia-network-operator



## 6. Install Network Operator Bundle - Non-published

Install operator-sdk as instructed [here](https://sdk.operatorframework.io/docs/installation/#install-from-github-release)

Install bundle:

```bash
operator-sdk run bundle --namespace nvidia-network-operator <bundleimage>
```

This will create CatalogSource, OperatorGroup, Subscription and will approve the InstallPlan.

Example of output:
```bash
INFO[0015] Successfully created registry pod: quay-io-frollandnvidia-network-operator-latest 
INFO[0015] Created CatalogSource: nvidia-network-operator-catalog 
INFO[0015] OperatorGroup "operator-sdk-og" created      
INFO[0015] Created Subscription: nvidia-network-operator-v1-1-0-sub 
INFO[0035] Approved InstallPlan install-h2xgn for the Subscription: nvidia-network-operator-v1-1-0-sub 
INFO[0035] Waiting for ClusterServiceVersion "nvidia-network-operator/nvidia-network-operator.v1.1.0" to reach 'Succeeded' phase 
INFO[0035]   Waiting for ClusterServiceVersion "nvidia-network-operator/nvidia-network-operator.v1.1.0" to appear 
INFO[0059]   Found ClusterServiceVersion "nvidia-network-operator/nvidia-network-operator.v1.1.0" phase: Pending 
INFO[0061]   Found ClusterServiceVersion "nvidia-network-operator/nvidia-network-operator.v1.1.0" phase: Installing 
INFO[0101]   Found ClusterServiceVersion "nvidia-network-operator/nvidia-network-operator.v1.1.0" phase: Succeeded 
INFO[0101] OLM has successfully installed "nvidia-network-operator.v1.1.0" 
```

## 7. Install Network Operator Bundle

See section: [Network Operator Installation Using CLI](https://docs.nvidia.com/networking/display/COKAN10/Network+Operator).

## 8. Install GPU Operator:

[GPU Operator docs](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/install-gpu-ocp.html#installing-the-nvidia-gpu-operator-using-the-cli)


## 9. Install SRIOV Operator

https://docs.openshift.com/container-platform/4.10/networking/hardware_networks/installing-sriov-operator.html