From https://github.com/Orange-OpenSource/towards5gs-helm

# How to use 

Pre-requirements:
- kubectl v1.20+ for SCTP support
- uname -r -> 5.0.0-23-generic or 5.4.x
- helm
- gtp5g kernel module on each worker node (see https://github.com/free5gc/gtp5g)

The best way to get all of these is to use the [quick-kubernetes](https://lwgitea.consrc.ca/hkerma/quick-kubernetes) repo.

# Deployment

```
curl https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml | kubectl create -f -
kubectl create namespace free5gc
```

From the host VM, create a `persistent_volume` on one of the worker node, for instance: `ssh k8s-node-1 mkdir -p /home/vagrant/persistent_volume`. Update the `persistent_volume.yaml` to replace the `nodeAffinity` value with the name of the worker node you chose:
```
nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k8s-node-1
```

If a different node or path is chosen, you need to update `persistent_volume.yaml` as well. Then go back to the master node and run

```
kubectl apply -f persistent_volume.yaml
```

If you used the quick-kubernetes deployment, each Kubernetes Node has 2 network interfaces: 
- `eth1` is a host-only adapter, meaning that Nodes can communicate together on this interface. Addresses are `192.168.50.10` for `k8s-master` and `192.168.50.(10+N)` for each `k8s-node-N`. It should be the main interface from communications between Nodes.
- `eth0` is a NAT adapter. It has Internet access and will be used as Data Netork interface.

In `free5gc/charts/freegc/values.yaml`, you need to change the `n6` interface subnet IP and default gateway to match those of your Data Network interface (in my case eth0). To find the IP, `ip addr`. To find the gateway, `ip route show 0.0.0.0/0 dev eth0 | cut -d\  -f3`.

If you use different interfaces, modify `free5gc/charts/freegc/values.yaml` to match the different interfaces. More precisely, `n2`, `n3`, `n4` and `n9` should be the host-only interfaces whereas `n6` should be the NAT interface.

<img src="https://www.researchgate.net/publication/336693290/figure/fig2/AS:816470774784005@1571673210936/Reference-point-representation-of-the-5G-system-architecture-6_W640.jpg" />

Use this command to deploy the 5G core.

```
helm install free5gc -n free5gc free5gc/charts/free5gc
```
You can check the state of the pods as well as the logs to make sure everything works correctly.

Go on the WEBUI (port 30500 on any Node) and add a new subscriber with the default values.
Credentials: admin/free5gc

Use this command to the deploy the UE RAN simulator.

```
helm install ueransim -n free5gc free5gc/charts/ueransim
```

You should then see in the gNB logs

```
[2021-11-25 23:37:44.853] [sctp] [info] Trying to establish SCTP connection... (10.100.50.249:38412)
[2021-11-25 23:37:44.860] [sctp] [info] SCTP connection established (10.100.50.249:38412)
```

NOTE: I had a big issue involving the VirtualBox host network, see [here](https://github.com/Orange-OpenSource/towards5gs-helm/issues/12). In summary, be careful when choosing your network interface in VBox and don't forget to enable "promiscuous mode: allow-all" as Multus-CNI uses macvlan to manage the 5G core's network interfaces.

NOTE 2: Enable [IP forward](https://github.com/Orange-OpenSource/towards5gs-helm/blob/main/docs/demo/Setup-free5gc-and-test-with-UERANSIM.md#tun-interface-correctly-created-on-the-ue-but-internet) so UE can reach the DN. With Calico, [it can be done](https://docs.projectcalico.org/reference/host-endpoints/forwarded) by editing calico configuration and adding `container_settings.allow_ip_forwarding: true`. If you use `quick-kubernetes`, it is enable automatically. Check if it is enables in the UPF pod by running `cat /proc/sys/net/ipv4/ip_forward`. It should return `1`.