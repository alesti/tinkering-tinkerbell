## K3D approach -- DRAFT 

I found in the [CNCF slack,
\#tinkerbell](https://cloud-native.slack.com/archives/C01SRB41GMT/p1672755969841129?thread_ts=1672423115.308059&cid=C01SRB41GMT)
a interesting start point: Using [k3d](https://k3d.io/v5.4.6/) - its a k3s in
docker - to run a local cluster without the k3s loadbalancer:

[K3d](https://k3d.io/) is a docker wraper for [k3s](https://k3s.io/) to run a
k8s cluster in a single docker container (ouch).  The difference to similar
local k8s clusters like [minikube](https://minikube.sigs.k8s.io/docs/start/) is
that k3d is reachable from outside of the node.

Tinkerbell does not use the default service [loadbalancer as it uses not only
tcp but also udp (bootp/tftp/rsyslogd)](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#design-details).

```bash
k3d cluster
create --network host --no-lb --k3s-arg "--disable=traefik,servicelb" --k3s-arg
"--kube-apiserver-arg=feature-gates=MixedProtocolLBService=true"
--host-pid-mode
```

Throws an error on raspi 4:

```bash
INFO[0022] Starting Node 'k3d-k3s-default-server-0'
WARN[0024] warning: encountered fatal log from node k3d-k3s-default-server-0 (retrying 0/10): Ptime="2023-01-22T15:41:09Z" level=fatal msg="failed to find memory cgroup (v2)" 
```

This needs to enable cgroups in the boot command by [adding the cgroup_memory=1
cgroup_enable=memory to
/boot/cmdline.txt](https://github.com/k3s-io/k3s-ansible/issues/179#issuecomment-1068384719).

```
[0] % k cluster-info
Kubernetes control plane is running at https://0.0.0.0:6443
CoreDNS is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

### Deploy Tinkerbell

And then deploy the tinkerbell stuff with
[helm](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#tldr) after some finetuning:

I needed to change the ip addresses to the one the k3d cluster is listening on
(192.168.48.5 in my case) in 
* [boots/values.yaml](https://github.com/tinkerbell/charts/blob/main/tinkerbell/boots/values.yaml)
  but do not change line 7, it needs to listen on 0.0.0.0 here to answer to
broadcasts from new nodes begging for pxe info.
* [stack/values.yaml](https://github.com/tinkerbell/charts/blob/main/tinkerbell/stack/values.yaml)
* its also neccessary to update the rufio version to [latest -
  sha-dee5b26](https://quay.io/repository/tinkerbell/rufio?tab=tags&tag=sha-dee5b26)
if you are on arm64, see [issue](https://github.com/tinkerbell/rufio/issues/74)

In general the versions in the `stack/values.yaml` are very old.

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

### Boots

Boots answers bootp requests, and it looks like that the clients get their ip:
  
```bash
{"level":"info","ts":1673884379.6893282,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:105","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"recv","mac":"00:1e:06:45:0d:48","via":"0.0.0.0","iface":"eth0","xid":"\"59:b6:c1:73\"","type":"DHCPDISCOVER"}
{"level":"info","ts":1673884379.689472,"caller":"boots/dhcp.go:88","msg":"parsed option82/circuitid","service":"github.com/tinkerbell/boots","pkg":"main","mac":"00:1e:06:45:0d:48","circuitID":""}
{"level":"info","ts":1673884379.6905186,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:61","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"send","mac":"00:1e:06:45:0d:48","dst":"255.255.255.255","iface":"eth0","xid":"\"59:b6:c1:73\"","type":"DHCPOFFER","address":"192.168.49.11","next_server":"192.168.49.2","filename":"http://192.168.49.2/ipxe/ipxe.efi"}
```

Something is missing: 

```bash
boots-5445bcd-ts7h9 boots {"level":"info","ts":1674409808.0051959,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.11 facility=daemon severity=ERR app-name=3218e9a030d5 procid=843 msg=\"{\\\"level\\\":\\\"info\\\",\\\"ts\\\":1674409823.155482,\\\"caller\\\":\\\"cmd/root.go:133\\\",\\\"msg\\\":\\\"no config file found\\\",\\\"service\\\":\\\"github.com/tinkerbell/tink\\\"}\\n\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
boots-5445bcd-ts7h9 boots {"level":"info","ts":1674409808.0052958,"caller":"syslog/receiver.go:113","msg":"host=192.168.48.11 facility=daemon severity=ERR app-name=3218e9a030d5 procid=843 msg=\"{\\\"level\\\":\\\"info\\\",\\\"ts\\\":1674409823.155634,\\\"caller\\\":\\\"cmd/root.go:45\\\",\\\"msg\\\":\\\"starting\\\",\\\"service\\\":\\\"github.com/tinkerbell/tink\\\",\\\"version\\\":\\\"9c098c1\\\"}\\n\"","service":"github.com/tinkerbell/boots","pkg":"syslog"}
```

In my ubuntu template is an url `IMG_URL: "http://192.168.48.5:8080/focal-server-cloudimg-amd64.raw.gz"`
I checked the endpoints, 8080 is connected to the tink-stack pod, and it has its webserver docroot at `/usr/share/nginx/html` with some files:

```
root@tink-stack-c7c964666-cl7p6:/usr/share/nginx/html# ls -la
total 432224
drwxr-xr-x 2 root root      4096 Jan 22 17:23 .
drwxr-xr-x 3 root root      4096 Oct  5 03:25 ..
-rw-r--r-- 1 root root       586 Jan 22 17:22 checksums.txt
-rw-r--r-- 1 1001  121 202634521 Apr 18  2017 initramfs-aarch64
-rw-r--r-- 1 1001  121 195462967 Apr 18  2017 initramfs-x86_64
-rw-r--r-- 1 1001  121  32971264 Apr 18  2017 vmlinuz-aarch64
-rw-r--r-- 1 1001  121  11504928 Apr 18  2017 vmlinuz-x86_64
``` 

I can get them:

```bash
[0] % wget http://192.168.48.5:8080/checksums.txt
--2023-01-22 19:12:00--  http://192.168.48.5:8080/checksums.txt
Verbindungsaufbau zu 192.168.48.5:8080 … verbunden.
HTTP-Anforderung gesendet, auf Antwort wird gewartet … 200 OK
Länge: 586 [text/plain]
Wird in »checksums.txt« gespeichert.

checksums.txt                                               100%[================================================================================================================================================>]     586  --.-KB/s    in 0s
```

 
