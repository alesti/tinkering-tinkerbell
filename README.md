# tinkering-tinkerbell
Tinkering around with [https://docs.tinkerbell.org/](tinkerbell)

Exploring bare metal k8s cluster provisioning...

## Hardware

### Nodes

I bought three [https://www.hardkernel.com/shop/odroid-h3/](Hardkernel Odroid
H3); this is a x86_64 single board computer with good specs (pxe booting, 2
network interfaces, changeble RAM and disk options).

First i aimed for arm64 (also known as raspi4, but the 8GB RAM version is more
expensive (if in stock somewhere at all)), and its not possible to beef them up
later. If Raspi is an option, head over to
[https://anthonynsimon.com/blog/kubernetes-cluster-raspberry-pi/](Building a
bare-metal Kubernetes cluster on Raspberry Pi).

My Odroids have (right now) 8 GB Ram and 256 GB NVME disk.

I am already familar with the H2, my personal one serves all typical local things at home, and we used
Odroid H2s to build an
[https://photos.google.com/share/AF1QipPIxF5isLFw8q3Y5bL6p22sNWmxLYC7JQUArTgIg4MjGRWVMu8LyGeXqT3R3Gx_gA?key=Z1ZZc3Z1bnAxakNpbEdfRTFLbk5TWDRBNXRUal93](18
node bare metal lab cluster on at work).

# Switch / Router

To reduce the blast radius i searched for a little router to separate the home
network from the cluster network.  I opted for a
[https://mikrotik.com/product/RB750Gr3](MikroTik hEX RB750Gr3). Its software
allows a lot of different configurations if neccessary.  I use one port as
uplink into the home network.

My H2 is dual homed, one leg in the home network, the other in the lab network.
On this leg it should act as provisioner for the cluster.

# Cooling

My H2 has a pwm controlled 92mm fan on the top of its case, but it barely uses
it under normal work load.  

I decided to use 80mm [https://noctua.at/en/nf-a8-pwm](Noctua NF-A8 PWM) fans,
suitable to the standard 4pin 2,54mm pitch connector.

The case (see below) is somehow size constrained, so only 80 mm fans are possible.

# Power Supply

The Odroids drain up to 4 Amps with two SATA disks and full cpu load with 15V DC, the MikroTik is also fine with 15V, so i bought a
[https://www.komerci.de/shop/stromversorgung/Festspannungsnetzgeraete/ps30swiv-festspannungsnetzgeraet-13-8v-30a-lcd](ham
radio power supply) with 15V and 30 Amps max) and some 5.5 mm dc power jacks with open wires and build a suiable power distribution cable.

### Case 

I printed a case with parts of
[https://www.thingiverse.com/thing:3485530](Odroid H2 Rackmount project) - they
have the same physical specs as their predecessor H2.

I needed to burn a lot of smaller filament remains, so it has really ugly colours :-)
[https://photos.google.com/share/AF1QipOEYq0544IV67harl58_uC0024xNleLqJeiRTEjn7_saC3fTc6Ne1Pnuho2mmJ2EA?key=SUhpWUtIOFYzX0pybnV2RXV3aVNjRk9uWXVsazFR](Odroid Cluster) 

This case design is so genius while it is not only endless stackable but it
has also cages/bays and caddies, the Odroids are mounted to the caddies and
removable from the rack. Thats not the case (sic!) at our company labcluster
case, it is really a mess to change a node, if neccessary!

The switch is just 1mm to big to fit into a cage, thats why it lives on the top
without the vertical columns.

I used 3mm rod to connect the units, the holes are for much bigger rod, so i
printed some spacers and pressed them into the connections between the units
([https://photos.google.com/share/AF1QipOEYq0544IV67harl58_uC0024xNleLqJeiRTEjn7_saC3fTc6Ne1Pnuho2mmJ2EA/photo/AF1QipOLIP7ZdU2PIErlum0OlAI_0ENNHN7T6_IcpPRl?key=SUhpWUtIOFYzX0pybnV2RXV3aVNjRk9uWXVsazFR](these blue tubes)).

The fan base is the longest thing i ever printed, it uses the whole printspace
my printer is able to serve (25cm). I melted threaded inserts into it to mount
the fans on it.
