# fosdem2024-sdn
Demo scripts / config for the "One SDN to rule them all" talk at FOSDEM 2024

## Simple overlay scenario
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
