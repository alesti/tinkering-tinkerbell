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
192.168.49.0/24 and is part of the lab net.  I preserve two ip (192.168.49.0 and .1)
to be not assigned by docker (to use it as connection from the host into the
cluster network).

```bash
docker network create -d macvlan --subnet=192.168.48.0/20 \
  --ip-range=192.168.49.0/24 --aux-address 'netaddress=192.168.49.0' \
  --aux-address 'shim=192.168.49.1' --gateway=192.168.48.1 \
  -o parent=enp2s0 clusternet
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
ping 192.168.49.2
ssh 192.168.49.2
docker exec -it lalala ash
```

### Start k3d cluster

[K3d](https://k3d.io/) is a docker wraper for [k3s](https://k3s.io/) to run a
k8s cluster in a single docker container (ouch).  The difference to similar
local k8s clusters like [minikube](https://minikube.sigs.k8s.io/docs/start/) is
that k3d is reachable from outside of the node.

Tinkerbell does not use the default service [loadbalancer as it uses not only
tcp but also udp (bootp/tftp/rsyslogd)](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#design-details).

```bash
k3d cluster create --network clusternet --no-lb \
  --k3s-arg "--disable=traefik,servicelb" \
  --k3s-arg "--kube-apiserver-arg=feature-gates=MixedProtocolLBService=true" \
   --host-pid-mode tinkerbell
```

Its api server was not reachable until i fetched the ip address by `docker
container inspect k3d-tinkerbell-server-0` and change the apiserver ip in
`~/.kube/config` to it with port 6443:

```bash
[0] % grep 192.168.49 ~/.kube/config
    server: https://192.168.49.2:6443

[1] % k cluster-info
Kubernetes control plane is running at https://192.168.49.2:6443
CoreDNS is running at https://192.168.49.2:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://192.168.49.2:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

### Deploy Tinkerbell

And then deploy the tinkerbell stuff with
[helm](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#tldr) after some finetuning:

I needed to change the ip addresses to the one the k3d cluster is listening on (192.168.49.2 in my case) in 
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

### Templates 

Creating templates for hardware and stuff: https://github.com/tinkerbell/sandbox#next-steps 

My templates are [there](configs/templates). 
I can load them into the cluster: 

```bash
[130] % for i in `ls -1`; do k apply -f $i ; done
hardware.tinkerbell.org/node-01 configured
hardware.tinkerbell.org/node-02 unchanged
hardware.tinkerbell.org/node-03 unchanged
template.tinkerbell.org/debian unchanged
template.tinkerbell.org/ubuntu-focal unchanged
workflow.tinkerbell.org/wf-node-01 unchanged
```


### Boots errors

Boots answers bootp requests, and it looks like that the clients get their ip:
  
```bash
{"level":"info","ts":1673884379.6893282,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:105","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"recv","mac":"00:1e:06:45:0d:48","via":"0.0.0.0","iface":"eth0","xid":"\"59:b6:c1:73\"","type":"DHCPDISCOVER"}
{"level":"info","ts":1673884379.689472,"caller":"boots/dhcp.go:88","msg":"parsed option82/circuitid","service":"github.com/tinkerbell/boots","pkg":"main","mac":"00:1e:06:45:0d:48","circuitID":""}
{"level":"info","ts":1673884379.6905186,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:61","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"send","mac":"00:1e:06:45:0d:48","dst":"255.255.255.255","iface":"eth0","xid":"\"59:b6:c1:73\"","type":"DHCPOFFER","address":"192.168.49.11","next_server":"192.168.49.2","filename":"http://192.168.49.2/ipxe/ipxe.efi"}
```

But the node [shows this error (screenshot)](pics/pxe-error.png):

```
Station IP address is 192.168.49.11

Server IP address is 192.168.49.2
NBP filename is ipxe.efi
NBP filesize is 0 Bytes
PXE-E99: Unexpected network error.
```

I installed a tcpdump in the boots pod, but i cannot see the given ip address there, but only in the boots logs above.

```
/ # tcpdump -n -i eth0 host 192.168.49.2 and not port 6443
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:59:32.644890 ARP, Reply 192.168.49.2 is-at 02:42:c0:a8:31:02, length 28
16:59:35.429925 IP 192.168.49.2.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, length 340
16:59:35.677055 ARP, Request who-has 192.168.49.2 (ff:ff:ff:ff:ff:ff) tell 192.168.49.2, length 28
16:59:38.708686 ARP, Reply 192.168.49.2 is-at 02:42:c0:a8:31:02, length 28
16:59:39.669119 IP 192.168.49.2.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, length 352
16:59:41.736753 ARP, Request who-has 192.168.49.2 (ff:ff:ff:ff:ff:ff) tell 192.168.49.2, length 28
16:59:43.552138 IP 192.168.49.2.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, length 352
16:59:43.601186 IP 192.168.49.2.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, length 352
16:59:44.792774 ARP, Reply 192.168.49.2 is-at 02:42:c0:a8:31:02, length 28
16:59:47.613377 IP 192.168.49.2.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, length 352
```

A filter on `host 192.168.49.11` does not show any traffic.  To be sure i
booted [grml](https://grml.org/) on the node, made a dhcp request, got the ip and was able to
download the ipxe.efi with `wget http://192.168.49.2/ipxe/ipxe.efi` as expected. 

I am running out of ideas right know, so i switched to another node (but same problem) and 
created a pcap file in boots to have a look with wireshark - it ran until the node reached the BIOS.
You can find it [here](configs/boots-tcpdump.out) to have a look.

For me it looks like the node gets [the DHCP
Offer (screenshot)](pics/boots-dhcp-offer.png) and it 'knows' its ip in the boot
screen, but it does not configure the interface accordingly - there is no
gratious arp or a connection attempt to the http server.

