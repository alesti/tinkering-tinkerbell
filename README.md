# tinkering tinkerbell
Tinkering around with [Tinkerbell](https://docs.tinkerbell.org/).
Exploring bare metal k8s cluster provisioning...

I want to learn more about the state of art kubernetes cluster provisoning on bare metal.

At work we use [Gardener](https://gardener.cloud/) to provision k8s clusters on
different providers as GCP, AWS, OpenStack, VMware.

Our companys (old, legacy, now switched off) bare metal cluster was provisioned by
terraform, metallb, matchbox, some outdated docker-images and a lot of scripts.

Nowadays there is the shiny [Cluster API](https://cluster-api.sigs.k8s.io/) to
provide the same declarative way to structure and provide nodes and
infrastructure as to k8s clusters itself, you might want to read [Cluster API -
A Guide on How to Get
Started](https://medium.com/condenastengineering/clusterapi-a-guide-on-how-to-get-started-ff9a81262945)
if you are new to that topic.

As far as i see, CAPI focuses on hyperscalers (as also Gardener does). 
I am interested in bare metal hardware provisoning.

Equinix Metal developed [Tinkerbell](https://docs.tinkerbell.org) and opensourced it, i will give it a try.

There are some shortcuts as [K3S](https://k3s.io/), but i want to use a small, container centric os like 
[flatcar](https://www.flatcar.org/), not a fullsize ubuntu, found 2 links to check later:

* https://tinkerbell.org/examples/flatcar-container-linux/
* https://kinvolk.io/blog/2020/10/provisioning-flatcar-container-linux-with-tinkerbell/

## Hardware

Find more about the used [hardware](hardware.md)

## Goofing around

[The WIP, just my notes to find a working solution](goofing-around.md).
  

