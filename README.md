# fosdem2024-sdn
Demo scripts / config for the
[One SDN to connect them all](https://fosdem.org/2024/schedule/event/fosdem-2024-1708-one-sdn-to-connect-them-all)
talk at FOSDEM 2024.

## Requirements
- git
- jq
- kind
- [OVN-Kubernetes CNI](https://github.com/ovn-org/ovn-kubernetes/)
- [Multus-cni](https://github.com/k8snetworkplumbingwg/multus-cni)
- [kubernetes-nmstate](https://github.com/nmstate/kubernetes-nmstate/)
- [KubeVirt](https://github.com/kubevirt/kubevirt)

## Setup demo environment
Refer to the OVN-Kubernetes repo KIND documentation for more details; the gist
of it is you should clone the OVN-Kubernetes repository, and run their kind
helper script:

```bash
git clone git@github.com:ovn-org/ovn-kubernetes.git

cd ovn-kubernetes
pushd contrib ; ./kind.sh --multi-network-enable -ic; popd
```

This will get you a running kind cluster (one control plane, and two worker
nodes) in interconnect mode, configured to use OVN-Kubernetes as the default
cluster network, configuring the multi-homing OVN-Kubernetes feature gate, and
deploying `multus-cni` in the cluster.

We will now need to patch the worker nodes, and install `NetworkManager` in
them (both are requirements for nmstate, which `kubernetes-nmstate` depends on).
```bash
for node in "ovn-worker" "ovn-worker2"; do
    kubectl label node "$node" node-role.kubernetes.io/worker=""

    "$OCI_BIN" exec "$node" bash -c "apt-get update && apt-get install network-manager -y && service NetworkManager start"
done
```

Once these steps are done, we can proceed to install kubernetes-nmstate; please
run the following commands:
```bash
kubectl apply -f https://github.com/nmstate/kubernetes-nmstate/releases/download/v0.81.0/nmstate.io_nmstates.yaml
kubectl apply -f https://github.com/nmstate/kubernetes-nmstate/releases/download/v0.81.0/namespace.yaml
kubectl apply -f https://github.com/nmstate/kubernetes-nmstate/releases/download/v0.81.0/service_account.yaml
kubectl apply -f https://github.com/nmstate/kubernetes-nmstate/releases/download/v0.81.0/role.yaml
kubectl apply -f https://github.com/nmstate/kubernetes-nmstate/releases/download/v0.81.0/role_binding.yaml
kubectl apply -f https://github.com/nmstate/kubernetes-nmstate/releases/download/v0.81.0/operator.yaml

cat <<EOF | kubectl create -f -
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
EOF

kubectl rollout status -w -n nmstate deployment nmstate-webhook --timeout=2m
```

Finally we will install KubeVirt; for that run the following commands:
```bash
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml"
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml"

kubectl -n kubevirt wait kv kubevirt --timeout=360s --for condition=Available
kubevirt.kubevirt.io/kubevirt condition met
```

Now we have everything we need to proceed with the demo.

## Simple overlay scenario
This scenario is ilustrated by the following diagram:
![](assets/overlay-scenario.png)

We want to create pods in two separate namespaces, communicated over a secondary
overlay. Two of those pods (one in each namespace) will expose an HTTP
application on port 9000, while the pod named `podclient` will try to access
those applications over the secondary network overlay.

The first thing you need to do is to create the namespaces required for this
demo; execute the following command:
```bash
for ns_name in "tenantblue" "tenantyellow"; do
    kubectl create namespace "$ns_name"
done
```

Afterwards, provision the manifests for this scenario:
- [network configuration](manifests/overlay/01-netconfig.yaml)
- [workloads](manifests/overlay/02-workloads.yaml)

You should now assure the pods enter the `Running` status:
```bash
kubectl wait pod --all --for=condition=Ready --namespace=tenantblue
kubectl wait pod --all --for=condition=Ready --namespace=tenantyellow
```

Once the pods are running, we should inspect the server pods to learn their IP
addresses:
```bash
kubectl get pods -ntenantyellow yellow-server \
    -ojsonpath="{@.metadata.annotations.k8s\.v1\.cni\.cncf\.io\/network-status}" | jq
[
  {
    "name": "ovn-kubernetes",
    "interface": "eth0",
    "ips": [
      "10.244.2.8"
    ],
    "mac": "0a:58:0a:f4:02:08",
    "default": true,
    "dns": {}
  },
  {
    "name": "tenantyellow/shared-net",
    "interface": "net1",
    "ips": [
      "192.168.200.7"
    ],
    "mac": "0a:58:c0:a8:c8:07",
    "dns": {}
  }
]

kubectl get pods -ntenantblue blue-server \
    -ojsonpath="{@.metadata.annotations.k8s\.v1\.cni\.cncf\.io\/network-status}" | jq
[
  {
    "name": "ovn-kubernetes",
    "interface": "eth0",
    "ips": [
      "10.244.2.7"
    ],
    "mac": "0a:58:0a:f4:02:07",
    "default": true,
    "dns": {}
  },
  {
    "name": "tenantblue/shared-net",
    "interface": "net1",
    "ips": [
      "192.168.200.4"
    ],
    "mac": "0a:58:c0:a8:c8:04",
    "dns": {}
  }
]
```

Knowing the IP address assigned by the CNI plugin, we can now access the
web-servers from the client pod:
```bash
kubectl exec -ntenantblue podclient -- curl 192.168.200.4:9000/hostname
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    11  100    11    0     0   9159      0 --:--:-- --:--:-- --:--:-- 11000
blue-server

kubectl exec -ntenantblue podclient -- curl 192.168.200.7:9000/hostname
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    13  100    13    0     0  26915      0 --:--:-- --:--:-- --:--:-- 13000
yellow-server
```

## Simple overlay scenario - mixed workloads without IPAM
We assume you've cleaned up your namespaces from the previous execution; if you
didn't, please do so now (delete & recreate the namespaces is the fastest way).

Once that is done, provision the manifests for this scenario:
- [network configuration](manifests/overlay/01-netconfig-no-ipam.yaml)
- [workloads](manifests/overlay/02-mixed-workloads.yaml)

The difference in this scenario is two-fold:
1. the network does **not** provide IPAM - this means the user must use static
IPs when defining the workloads
2. the scenario consists of both VM and pod worklods in the same platform

But the overall idea is the same; we want to curl both servers from the client
(this time it is a VM); for that, we will use an helper binary, and log in via
console:

```bash
virtctl console vm-client -ntenantblue # the credentials are "fedora" / "fedora"
Successfully connected to vm-server console. The escape sequence is ^]
vm-server login: fedora
Password:
[fedora@vm-server ~]$ curl 192.168.200.20:9000/hostname
blue-server[fedora@vm-server ~]$ curl 192.168.200.30:9000/hostname
yellow-server[fedora@vm-server ~]$
```
