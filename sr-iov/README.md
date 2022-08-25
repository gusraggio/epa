# SR-IOV Setup

Provision an RKE2 cluster with multus and canal.

## Check cards:
```
node2:~ # lshw -c network -businfo
Bus info          Device      Class          Description
========================================================
pci@0000:01:00.0  em1         network        Ethernet Controller 10-Gigabit X540-AT2
pci@0000:01:00.1  em2         network        Ethernet Controller 10-Gigabit X540-AT2
pci@0000:06:00.0  em3         network        I350 Gigabit Network Connection
pci@0000:06:00.1  em4         network        I350 Gigabit Network Connection
node2:~ #
```
### Check which cards are SR-IOV Capable

```
node2:~ # lspci -vs 0000:01:00.0
01:00.0 Ethernet controller: Intel Corporation Ethernet Controller 10-Gigabit X540-AT2 (rev 03)
	Subsystem: Dell Ethernet 10G 4P X540/I350 rNDC
	Flags: bus master, fast devsel, latency 0, IRQ 53, NUMA node 0
	Memory at 91c00000 (64-bit, prefetchable) [size=2M]
	Memory at 91e04000 (64-bit, prefetchable) [size=16K]
	Expansion ROM at 92400000 [disabled] [size=512K]
	Capabilities: [40] Power Management version 3
	Capabilities: [50] MSI: Enable- Count=1/1 Maskable+ 64bit+
	Capabilities: [70] MSI-X: Enable+ Count=64 Masked-
	Capabilities: [a0] Express Endpoint, MSI 00
	Capabilities: [e0] Vital Product Data
	Capabilities: [100] Advanced Error Reporting
	Capabilities: [150] Alternative Routing-ID Interpretation (ARI)
	Capabilities: [160] Single Root I/O Virtualization (SR-IOV)
	Capabilities: [1d0] Access Control Services
	Kernel driver in use: ixgbe
	Kernel modules: ixgbe

node2:~ #
``` 

## There are 2 ways of discovering the hardware: Auto Discovery or Manual

### Auto discovery with nfd

```
kubectl apply -k https://github.com/kubernetes-sigs/node-feature-discovery/deployment/overlays/default?ref=v0.11.2
```
Check if the capable nodes were labeled:

```
kubectl get node node1 -o json | grep feature.node.kubernetes.io/network-sriov.capable
```
```
            "feature.node.kubernetes.io/network-sriov.capable": "true"
```

### Manual

```
kubectl label node $node_name feature.node.kubernetes.io/network-sriov.capable=true
```

## Install SR-IOV chart 

Install the SR-IOV chart from the App menu in the cluster explorer in rancher

![rancher-apps](others/apps.png)

Check that the crds are present
```
kubectl get crd | grep openshift
sriovibnetworks.sriovnetwork.openshift.io            2022-04-22T09:37:55Z
sriovnetworknodepolicies.sriovnetwork.openshift.io   2022-04-22T09:37:55Z
sriovnetworknodestates.sriovnetwork.openshift.io     2022-04-22T09:37:55Z
sriovnetworkpoolconfigs.sriovnetwork.openshift.io    2022-04-22T09:37:55Z
sriovnetworks.sriovnetwork.openshift.io              2022-04-22T09:37:55Z
sriovoperatorconfigs.sriovnetwork.openshift.io       2022-04-22T09:37:55Z
```
to check if the VFs are there do:
```
kubectl get sriovnetworknodestates.sriovnetwork.openshift.io -A
```

for creating 4 VFs in node1 do:
```
kubectl apply -f node1-policy.yaml
```

for creating 4 VFs in node2 do:
```
kubectl apply -f node2-policy.yaml
```

# IMPORTANT! The namespace of the manifest must be the same where sriov-network-config-daemon and the sriov deployment are running

You should now see on your nodes your 4 VFs created on the selected interface
```
node1:~ # ip link show em1
4: em1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 24:6e:96:cd:15:18 brd ff:ff:ff:ff:ff:ff
    vf 0     link/ether 92:9e:4a:62:b3:8b brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 1     link/ether d2:5d:82:2e:94:c3 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 2     link/ether 1e:8f:7f:86:7e:e1 brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    vf 3     link/ether b6:58:af:5d:b7:cf brd ff:ff:ff:ff:ff:ff, spoof checking on, link-state auto, trust off, query_rss off
    altname eno1
    altname enp1s0f0
node1:~ #
```

The node should have now a new allocatable resource called rancher.io/intelnics. In my case:
```
kubectl get node node1 -o jsonpath='{.status.allocatable}' | jq
{
  "cpu": "32",
  "ephemeral-storage": "758911735828",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "4000Mi",
  "memory": "259480852Ki",
  "pods": "110",
  "rancher.io/intelnics": "4"
}
```

Now it is time to create the network that includes the previous resources. This will end up creating a net-attach resource for multus. For example:
```
kubectl apply -f networks.yaml
```

Again, the namespace must be the same as the previous ones. If it worked, we should see:
```
kubectl get network-attachment-definitions.k8s.cni.cncf.io -A
NAMESPACE     NAME              AGE
kube-system   example-network   11s
```
