## Goofing around (fast moving targets - DRAFT)

Tinkerbell bootstraps a single node cluster to run the tinkerbell services on it and work as provisioner for other hardware.

I tried the [sandbox approach with docker
compose](https://github.com/tinkerbell/sandbox/blob/main/docs/quickstarts/COMPOSE.md),
but the machine which i wanted to use as provision server has already too many
services on it which ports are also requested by the tinkerbell services (even
if i bind my services to a dedicated interface, docker tries to bind
0.0.0.0:<port> in many cases).

I was not able to bind all services to other ports in the docker-compose file, i got a k3s cluster but with only a coredns pod running.
I goofed around in the logfiles, but i lost faith and went to another way.

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

Or, if was already created: `k3d cluster start k3s-default`

FIXME: I am not sure if `--network host` is a clever option. It seems to use the other
       interface of my H2 (i think while it has the default gw in its network).
       Check https://k3d.io/v5.0.1/usage/commands/k3d_cluster_create/ options later to find a better(?) solution.



And then deploy the tinkerbell stuff with [helm](https://github.com/tinkerbell/charts/tree/main/tinkerbell/stack#tldr):

```bash
(in tinkerbell/charts/tinkerbell)
helm dependency build stack/
trusted_proxies=$(kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' | tr ' ' ',')
helm install stack-release stack/ --create-namespace --namespace tink-system --wait --set "boots.trustedProxies=${trusted_proxies}" --set "hegel.trustedProxies=${trusted_proxies}"
```

I am on that right now, and i can see one of my odroids screaming:

```bash
{"level":"info","ts":1673017452.6608596,"caller":"dhcp4-go@v0.0.0-20190402165401-39c137f31ad3/handler.go:105","msg":"","service":"github.com/tinkerbell/boots","pkg":"dhcp","pkg":"dhcp","event":"recv","mac":"00:1e:06:45:01:1e","via":"0.0.0.0","iface":"enp2s0","xid":"\"f1:4e:78:13\"","type":"DHCPDISCOVER","secs":28}
```

### Testing templates 

```
trusted_proxies=$(kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}' | tr ' ' ',')
helm template -f values.yaml . --dry-run --output-dir=tmp --set "boots.trustedProxies=${trusted_proxies}" --set "hegel.trustedProxies=${trusted_proxies}"
```

I changed all ip related values in
https://github.com/tinkerbell/charts/blob/main/tinkerbell/stack/values.yaml to
the ip address to 192.168.48.10 (provisioner host in cluster network) and set this ip also explicitly where the bind was only `:port` (any). Same for boots values (https://github.com/tinkerbell/charts/blob/main/tinkerbell/boots/values.yaml).


```bash
(ssh) aleks@odroid-1 ‹ main ↑●● › : ~/data/git/tinkering-tinkerbell/configs
[0] % TINKERBELL_HOST_IP=192.168.48.10 ; docker build -f Dockerfile_flatcar-install -t $TINKERBELL_HOST_IP/flatcar-install .
```

`docker push $TINKERBELL_HOST_IP/flatcar-install` failed as the registry has an unknown cert ca.

Ok, i stop here and try to get the k3d cluster running on the right ip first.




