# tinkering tinkerbell 

Tinkering around with [Tinkerbell](https://docs.tinkerbell.org/) - exploring bare metal k8s cluster
provisioning...

I want to learn more about the state of art kubernetes cluster provisoning on bare metal.

At work we use [Gardener](https://gardener.cloud/) to provision k8s clusters on
different (cloud)providers as GCP, AWS, OpenStack, VMware.

Our companys (old, legacy, now switched off) bare metal cluster was provisioned by
terraform, metallb, matchbox, some outdated docker-images and a lot of scripts.

Nowadays there is the shiny [Cluster API](https://cluster-api.sigs.k8s.io/) to
provide the same declarative way to structure and provide nodes and
other infrastructure as to k8s clusters itself, you might want to read [Cluster API -
A Guide on How to Get
Started](https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945)
if you are new to that topic.

As far as i see, CAPI focuses on hyperscalers (as also Gardener does). 
I am interested in bare metal hardware provisoning.

Equinix Metal developed [Tinkerbell](https://docs.tinkerbell.org) and opensourced it, i will give it a try.

There are some shortcuts as [K3S](https://k3s.io/), but i want to use a small, container centric os like 
[flatcar](https://www.flatcar.org/), not a fullsize ubuntu. 

In a first step i tried the [Tinkerbell sandbox](https://github.com/tinkerbell/sandbox#quick-starts).

See my latest notes in the [k3d approach with a dedicated master, AMD64 arch](k3d-approach-dedicated-master.md).

I tried (and wrote it down) different things before, but its more for me to
(later) understand what was going wrong: [k3d approach with docker on a
dedicated interface](k3d-approach-docker-dedicated-interface.md), [k3d approach
with a raspi](k3d-raspi-approach.md)(stopped by getting moar amd64 hardware).

## Hardware

I decided to use dedicated hardware for that experiment.

![2nd generation case with Odroids (H2 in black, H3 in colored rack)](pics/case-2nd-gen_sm.jpg)

Find more about the used [hardware](hardware.md).

## Resources

* Cluster API: https://medium.com/swlh/clusterops-1-line-commit-to-upgrade-your-kubernetes-clusters-de3548124d04
* Bare-Metal Chronicles - Tinkerbell: https://www.youtube.com/watch?v=NCFUUjTw6hA
* TGI k8s 158 - bare metal clusters with Cluster API Tinkerbell: https://www.youtube.com/watch?v=Di_AR6nAss0
* Tinkerbell with flatcar: https://tinkerbell.org/examples/flatcar-container-linux/
* Flatcar with Tinkerbell: https://kinvolk.io/blog/2020/10/provisioning-flatcar-container-linux-with-tinkerbell/
* Tink binaries https://github.com/tinkerbell/tink
