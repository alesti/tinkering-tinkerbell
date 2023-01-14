## K3D approach -- DRAFT 

I found in the [CNCF slack,
\#tinkerbell](https://cloud-native.slack.com/archives/C01SRB41GMT/p1672755969841129?thread_ts=1672423115.308059&cid=C01SRB41GMT)
a interesting start point: Using [k3d](https://k3d.io/v5.4.6/) - its a k3s in
docker - to run a local cluster without the k3s loadbalancer:

```bash
k3d cluster
create --network host --no-lb --k3s-arg "--disable=traefik,servicelb" --k3s-arg
"--kube-apiserver-arg=feature-gates=MixedProtocolLBService=true"
--host-pid-mode
```

### Bind docker to the right interface

I am not sure if `--network host` is a clever option, in my case (i use a dual
interface node) it seems to use the default interface of my Odroid H2 [see
hardware.md](hardware.md) which is packed with a lot of services i need at home
(piHole, homeassistant, grafana, influxdb, ...) which block a lot ports on that
interface.

I want to run the provisioner/tinkerbell stuff on the 2nd, so far unused
interface.  So i needed to figure out how to create a docker network and bind
it to an dedicated interface.  I used [dockers macvlan in
bridge-mode](https://docs.docker.com/network/macvlan/#bridge-mode), and i found
a blog post how to reach that docker network also from the node itself [using
docker macvlan
networks](https://blog.oddbit.com/post/2018-03-12-using-docker-macvlan-networks/)

I have a distinct network for the lab (192.168.48.0/20) (see
[hwconfig.md](configs/hwconfig.md)).  The new docker network will have
192.168.49.0/24 and is part of the lab net.  I preserve one ip (192.168.49.1)
to be not assigned by docker (to use it as connection from the host into the
cluster network).
This matroshka net will use the first ip (192.168.49.0) as
host ip, but i dont care.

```bash
docker network create -d macvlan --subnet=192.168.48.0/20 \
  --ip-range=192.168.49.0/24 --aux-address 'host=192.168.49.1' \
  --gateway=192.168.48.1 -o parent=enp2s0 clusternet
# have a look with
docker network inspect clusternet

# creating access from the node
ip link add clusternet-shim link enp2s0 type macvlan  mode bridge
ip addr add 192.168.49.1/32 dev clusternet-shim
ip link set clusternet-shim up
ip route add 192.168.49.0/24 dev clusternet-shim

# some tests 
docker run --rm -dit --network clusternet --name lalala alpine:latest ash
docker container inspect lalala
ping 192.168.48.2
ping 192.168.49.0
ssh 192.168.49.0
docker exec -it lalala ash
```

### Start k3d cluster

[K3d](https://k3d.io/) is a docker wraper for [k3s](https://k3s.io/) to run a
k8s cluster in a single docker container (ouch).  The difference to similar
local k8s clusters like [minikube](https://minikube.sigs.k8s.io/docs/start/) is
that k3d is reachable from outside of the node.

Tinkebell does not use the default service loadbalancer as it uses not only tcp.

```bash
k3d cluster create --network clusternet --no-lb \
  --k3s-arg "--disable=traefik,servicelb" \
  --k3s-arg "--kube-apiserver-arg=feature-gates=MixedProtocolLBService=true" \
   --host-pid-mode tinkerbell
```

Its api server is not reachable until i did some fine tuning. 
I fetched the ip address by `docker container inspect k3d-tinkerbell-server-0` and
change the apiserver ip in `~/.kube/config` to it with port 6443:

```bash
[0] % grep 192.168.49 ~/.kube/config
    server: https://192.168.49.0:6443

[1] % k cluster-info
Kubernetes control plane is running at https://192.168.49.0:6443
CoreDNS is running at https://192.168.49.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://192.168.49.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

### Deploy Tinkerbell

And then deploy the tinkerbell stuff with
[helm](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#tldr) after some finetuning:

I needed to change the ip addresses to the one the k3d cluster is listening on (192.168.49.0 in my case) in 
* [boots/values.yaml](https://github.com/tinkerbell/charts/blob/main/tinkerbell/boots/values.yaml)
  but do not change line 7, it needs to listen on 0.0.0.0 here to answer to broadcasts from new nodes begging for pxe info.
* [stack/values.yaml](https://github.com/tinkerbell/charts/blob/main/tinkerbell/stack/values.yaml)

See the [stack
README](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#installing-the-chart)
for some more details.
 
Then deploy it with helm (in [tinkerbell/charts/tinkerbell](https://github.com/tinkerbell/charts/tree/main/tinkerbell)) 

```bash
helm dependency build stack/
trusted_proxies=$(kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' | tr ' ' ',')
helm install stack-release stack/ --create-namespace --namespace tink-system \
  --wait --set "boots.trustedProxies=${trusted_proxies}" --set "hegel.trustedProxies=${trusted_proxies}"
```

After some time a new namespace (tink-system) appears and the tink pods.

```bash
[0] % k get po
NAME                               READY   STATUS    RESTARTS   AGE
kube-vip-524jp                     1/1     Running   0          173m
tink-server-6444c7d846-424zn       1/1     Running   0          173m
tink-controller-6b7d78d5d4-cc4fl   1/1     Running   0          173m
hegel-58b567d49d-fwhn8             1/1     Running   0          173m
rufio-798b6689b4-b4468             1/1     Running   0          173m
tink-stack-c7c964666-65dt2         1/1     Running   0          173m
boots-6c7fdc88bc-649gh             1/1     Running   0          58m
```

And, if i start one of my nodes, i can see in boots logs that it begs for help:

```bash
{"level":"info","ts":1673711412.2549672,"caller":"boots/dhcp.go:88","msg":"parsed option82/circuitid",
  "service":"github.com/tinkerbell/boots","pkg":"main","mac":"00:1e:06:45:01:1e","circuitID":""}
{"level":"error","ts":1673711412.2551427,"caller":"boots/dhcp.go:101","msg":"retrieved job is empty",
  "service":"github.com/tinkerbell/boots","pkg":"main","type":"DHCPDISCOVER","mac":"00:1e:06:45:01:1e",
  "error":"discover from dhcp message: no hardware found","errorVerbose":
```

### Next Step

Creating templates for hardware and stuff: https://github.com/tinkerbell/sandbox#next-steps 

:-) 
