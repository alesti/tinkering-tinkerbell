## Goofing around (fast moving targets - DRAFT - unstructured notes)

If i succeed i will write a runbook from these notes later.

Tinkerbell bootstraps a single node cluster (k3s), but runs the services in parallel next to it - why?

I tried the [sandbox approach with docker
compose](https://github.com/tinkerbell/sandbox/blob/main/docs/quickstarts/COMPOSE.md),
but the machine which i wanted to use as provision server has already too many
services on it which ports are also requested by the tinkerbell services (even
if i bind my services to a dedicated interface, docker tries to bind
0.0.0.0:<port> in many cases).

I was not able to bind all services to other ports in the docker-compose file,
i got a k3s cluster but with only a coredns pod running.  I goofed around in
the logfiles, but i lost faith and went to another way for the next round.
I came back as i solved the problem to bind the docker network to the 2nd interface, see below.

### prequsites for docker compose approach

Docker should be the ce version, if not the compose plugin is too old. I used
https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-22-04
to install it from docker and not from ubuntu repo, and `sudo apt-get install
docker-compose-plugin`.

Instead `docker compose up -d` i used `docker compose up &>/tmp/log` and tail
the logfile to get a clue whats going on.

It boots, and if i reset my 2nd node, it loads the LinuxKit preboot env, but:

```bash
compose-web-assets-server-1             | 2023/01/09 17:03:32 [error] 32#32: *4 open() "/usr/share/nginx/html/workflow/ca.pem" failed (2: No such file or directory), client: 192.168.48.11, server: localhost, request: "GET /workflow/ca.pem HTTP/1.1", host: "192.168.48.12:8080"
compose-web-assets-server-1             | 192.168.48.11 - - [09/Jan/2023:17:03:32 +0000] "GET /workflow/ca.pem HTTP/1.1" 404 153 "-" "Go-http-client/1.1" "-"
```

And yes, the whole dir `workflow` is not there:
```bash
[126] % docker exec -it compose-web-assets-server-1 sh
/ # cd /usr/share/nginx/html/
/usr/share/nginx/html # ls -la
total 1430476
drwxr-xr-x    2 root     root          4096 Jan  8 19:52 .
drwxr-xr-x    3 root     root          4096 Dec 14 01:20 ..
-rw-r--r--    1 root     root     605424786 Jan  8 19:52 focal-server-cloudimg-amd64.raw.gz
-rw-r--r--    1 root     root     211721656 Jan  8 19:52 hook_aarch64.tar.gz
-rw-r--r--    1 root     root     205051200 Jan  8 19:51 hook_x86_64.tar.gz
-rw-r--r--    1 1001     121      202634521 Apr 18  2017 initramfs-aarch64
-rw-r--r--    1 1001     121      195462967 Apr 18  2017 initramfs-x86_64
-rw-r--r--    1 1001     121       32971264 Apr 18  2017 vmlinuz-aarch64
-rw-r--r--    1 1001     121       11504928 Apr 18  2017 vmlinuz-x86_64
/usr/share/nginx/html #
```

I still do not get the cause of running a k3s, there is no tink-system ns nor any pod init, all dockers are running running separatly. 
FIXME: Ask in slack.

```bash
Every 2,0s: docker ps                                                                                                                            node-02: Mon Jan  9 18:10:09 2023

CONTAINER ID   IMAGE                                       COMMAND                  CREATED        STATUS                  PORTS
                                                                     NAMES
c0502c674679   quay.io/tinkerbell/tink-controller:v0.8.0   "/usr/bin/tink-contr…"   22 hours ago   Up 22 hours             42113-42114/tcp
                                                                     compose-tink-controller-1
4aa0e08633af   quay.io/tinkerbell/boots:v0.7.0             "/usr/bin/boots -log…"   22 hours ago   Up 22 hours
                                                                     compose-boots-1
684c14079a0e   quay.io/tinkerbell/tink:v0.8.0              "/usr/bin/tink-server"   22 hours ago   Up 22 hours (healthy)   0.0.0.0:42113-42114->42113-42114/tcp, :::42113-42114->4
2113-42114/tcp                                                       compose-tink-server-1
af81bb246257   quay.io/tinkerbell/hegel:v0.8.0             "/usr/bin/hegel"         22 hours ago   Up 22 hours             0.0.0.0:50060-50061->50060-50061/tcp, :::50060-50061->5
0060-50061/tcp                                                       compose-hegel-1
ae7d50fb5e7d   quay.io/tinkerbell/rufio:v0.1.0             "/manager --kubeconf…"   22 hours ago   Up 22 hours
                                                                     compose-rufio-1
db3c369abe6a   nginx:alpine                                "/docker-entrypoint.…"   22 hours ago   Up 22 hours             0.0.0.0:8080->80/tcp, :::8080->80/tcp
                                                                     compose-web-assets-server-1
178bce650b8d   rancher/k3s:v1.24.4-k3s1                    "/bin/k3s server --d…"   22 hours ago   Up 22 hours (healthy)   0.0.0.0:6443->6443/tcp, :::6443->6443/tcp, 0.0.0.0:8888
->80/tcp, :::8888->80/tcp, 0.0.0.0:8443->443/tcp, :::8443->443/tcp   compose-k3s-1
```

## creating image

Creating image: https://docs.tinkerbell.org/deploying-operating-systems/examples-ubuntu/

wget https://cloud-images.ubuntu.com/daily/server/focal/current/focal-server-cloudimg-amd64.img -O focal-server-cloudimg-amd64.img


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
Thats not easy to do, so i gave up and created a lab node with only one
interface (two workers might be enough to figure out how to setup tinkerbell on
my existing twolegged hardware)

