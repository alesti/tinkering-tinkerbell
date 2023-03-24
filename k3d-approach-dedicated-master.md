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

Not working for me - i got in struggle with that as i tried to use a dedicated interface for docker, see 
[k3d-approach-docker-dedicated-interface.md](k3d-approach-docker-dedicated-interface.md)

I using the main (with the default route on it) interface of a odroid as master node.

### Start k3d cluster

[K3d](https://k3d.io/) is a docker wraper for [k3s](https://k3s.io/) to run a
k8s cluster in a single docker container (ouch).  The difference to similar
local k8s clusters like [minikube](https://minikube.sigs.k8s.io/docs/start/) is
that k3d is reachable from outside of the node.

Tinkerbell does not use the default service [loadbalancer as it uses not only
tcp but also udp (bootp/tftp/rsyslogd)](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#design-details).

```bash
k3d cluster create --network host --no-lb \
  --k3s-arg "--disable=traefik,servicelb" \
  --k3s-arg "--kube-apiserver-arg=feature-gates=MixedProtocolLBService=true" \
   --host-pid-mode tinkerbell
```

[0] % k cluster-info
Kubernetes control plane is running at https://0.0.0.0:6443
CoreDNS is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

### Deploy Tinkerbell

And then deploy the tinkerbell stuff with
[helm](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#tldr) after some finetuning:

I needed to change the ip addresses to the one the k3d cluster is listening on (192.168.48.2 in my case) in 
* [boots/values.yaml](https://github.com/tinkerbell/charts/blob/main/tinkerbell/boots/values.yaml)
  but do not change line 7, it needs to listen on 0.0.0.0 here to answer to broadcasts from new nodes begging for pxe info.
* [stack/values.yaml](https://github.com/tinkerbell/charts/blob/main/tinkerbell/stack/values.yaml)

See the [stack
README](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#installing-the-chart)
for some more details.
 
Then deploy it with helm (in [tinkerbell/charts/tinkerbell](https://github.com/tinkerbell/charts/tree/main/tinkerbell)):

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
[...]
template.tinkerbell.org/debian configured
template.tinkerbell.org/ubuntu-focal configured
workflow.tinkerbell.org/wf-node-01 configured
```

Now there are the kinds hardware, template and workflow in the cluster, in my case in ns default.

### Boots 

Boots answers bootp requests, and it looks like that the clients get their ip:
  
```bash
{"level":"info","ts":1679670723.592032,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:105","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"recv","mac":"00:1e:06:45:0d:87","via":"0.0.0.0","iface":"enp2s0","xid":"\"6d:fa:34:df\"","type":"DHCPDISCOVER"}
{"level":"info","ts":1679670723.5944743,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:61","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"send","mac":"00:1e:06:45:0d:87","dst":"255.255.255.255","iface":"enp2s0","xid":"\"6d:fa:34:df\"","type":"DHCPOFFER","address":"192.168.48.14","next_server":"192.168.48.2","filename":"ipxe.efi"}
{"level":"info","ts":1679670727.298966,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:105","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"recv","mac":"00:1e:06:45:0d:87","via":"0.0.0.0","iface":"enp2s0","xid":"\"6d:fa:34:df\"","type":"DHCPREQUEST"}
```

It tries to send the ipxe.efi file, but thats unsuccessful:

```bash
{"level":"info","ts":1679670727.3009703,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:61","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"send","mac":"00:1e:06:45:0d:87","dst":"255.255.255.255","iface":"enp2s0","xid":"\"6d:fa:34:df\"","type":"DHCPACK","address":"192.168.48.14","next_server":"192.168.48.2","filename":"ipxe.efi"}
{"level":"error","ts":1679670727.305437,"logger":"github.com/tinkerbell/ipxedust","caller":"itftp/itftp.go:97","msg":"file serve failed","service":"github.com/tinkerbell/boots","event":"get","filename":"ipxe.efi","uri":"ipxe.efi","client":{"IP":"192.168.48.14","Port":1030,"Zone":""},"macFromURI":"","b":0,"contentSize":1017856,"error":"sending block 0: code=8, error: User aborted the transfer\u0000","stacktrace":"github.com/tinkerbell/ipxedust/itftp.Handler.HandleRead\n\t/home/runner/go/pkg/mod/github.com/tinkerbell/ipxedust@v0.0.0-20220908192154-99b8049fc267/itftp/itftp.go:97\ngithub.com/pin/tftp/v3.(*Server).handlePacket.func2\n\t/home/runner/go/pkg/mod/github.com/pin/tftp/v3@v3.0.0/server.go:440"}
```

It tries now auto.ipxe and this works (i am not sure if it is a proper way to simply use another boot image):

```bash
{"level":"info","ts":1679670750.3811285,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:61","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"send","mac":"00:1e:06:45:0d:87","dst":"255.255.255.255","iface":"enp2s0","xid":"\"a3:82:6e:72\"","type":"DHCPOFFER","address":"192.168.48.14","next_server":"192.168.48.2","filename":"http://192.168.48.2/auto.ipxe"}
{"level":"info","ts":1679670750.383501,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:105","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"recv","mac":"00:1e:06:45:0d:87","via":"0.0.0.0","iface":"enp2s0","xid":"\"a3:82:6e:72\"","type":"DHCPREQUEST","secs":10}
{"level":"info","ts":1679670750.3891742,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:61","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"send","mac":"00:1e:06:45:0d:87","dst":"255.255.255.255","iface":"enp2s0","xid":"\"a3:82:6e:72\"","type":"DHCPACK","address":"192.168.48.14","next_server":"192.168.48.2","filename":"http://192.168.48.2/auto.ipxe"}
{"level":"info","ts":1679670754.3173676,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: ... ok\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670754.3175762,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: net0: 192.168.48.14/255.255.240.0 gw 192.168.48.1\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670754.317983,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: Next server: 192.168.48.2\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670754.318064,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: Filename: http://192.168.48.2/auto.ipxe\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"debug","ts":1679670754.3179936,"caller":"httplog/httplog.go:29","msg":"","service":"github.com/tinkerbell/boots","pkg":"http","event":"sr","method":"GET","uri":"/auto.ipxe","client":"192.168.48.14"}
{"level":"info","ts":1679670754.3190768,"caller":"job/job.go:151","msg":"discovering from ip","service":"github.com/tinkerbell/boots","ip":"192.168.48.14"}
{"level":"info","ts":1679670754.3201606,"caller":"httplog/httplog.go:37","msg":"","service":"github.com/tinkerbell/boots","pkg":"http","event":"ss","method":"GET","uri":"/auto.ipxe","client":"192.168.48.14","duration":0.002213165,"status":200}
{"level":"info","ts":1679670754.3224542,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: http://192.168.48.2/auto.ipxe... ok\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670754.3234913,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: auto.ipxe : 1110 bytes [script]\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
```

It checks, what and how to provision and sends a vmlinux-x86_64.img and an initramfs, the nodes starts to boot, but something is wrong or lacks info:

```bash
{"level":"debug","ts":1679670754.3235824,"caller":"httplog/httplog.go:29","msg":"","service":"github.com/tinkerbell/boots","pkg":"http","event":"sr","method":"POST","uri":"/phone-home","client":"192.168.48.14"}
{"level":"info","ts":1679670754.3236086,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: Tinkerbell Boots iPXE\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670754.3237169,"caller":"job/job.go:151","msg":"discovering from ip","service":"github.com/tinkerbell/boots","ip":"192.168.48.14"}
{"level":"info","ts":1679670754.3241332,"caller":"job/events.go:120","msg":"proxied event","service":"github.com/tinkerbell/boots","mac":"00:1e:06:45:0d:87","hardware.id":"00:1e:06:45:0d:87","instance.id":"00:1e:06:45:0d:87","kind":"provisioning.104.01"}
{"level":"info","ts":1679670754.3241878,"caller":"job/events.go:45","msg":"updated allow_pxe","service":"github.com/tinkerbell/boots","mac":"00:1e:06:45:0d:87","hardware.id":"00:1e:06:45:0d:87","instance.id":"00:1e:06:45:0d:87","allow_pxe":false}
{"level":"info","ts":1679670754.3243015,"caller":"httplog/httplog.go:37","msg":"","service":"github.com/tinkerbell/boots","pkg":"http","event":"ss","method":"POST","uri":"/phone-home","client":"192.168.48.14","duration":0.00071725,"status":200}
{"level":"info","ts":1679670754.3247502,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: http://192.168.48.2/phone-home... ok\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670754.4262772,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: http://192.168.48.2:8080/vmlinuz-x86_64... ok\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
{"level":"info","ts":1679670756.138107,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.14 facility=kern severity=INFO app-name=node-04 msg=\" ipxe: http://192.168.48.2:8080/initramfs-x86_64... ok\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
```

This looks on the client like this [screenshot](pics/node-04-linuxkit.png)

