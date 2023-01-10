## Hardware

### Nodes

I bought three [Hardkernel Odroid
H3](https://www.hardkernel.com/shop/odroid-h3/); a x86_64 single board computer
with good specs (pxe booting, 2 network interfaces, changeble RAM and multiple
disk options).

First i aimed for arm64 (also known as Raspberry4, but the 8GB RAM version is
at 180 € right now, if it is in stock somewhere, and its not possible
to add more RAM later. If Raspi is an option, you should head over to [Building a
bare-metal Kubernetes Cluster on Raspberry
Pi](https://anthonynsimon.com/blog/kubernetes-cluster-raspberry-pi/) and https://github.com/christianhuening/pinetes/blob/main/README.md.

My Odroids have (right now) 8 GB Ram and 256 GB NVME disk.

I am already familar with the H2, my personal one serves all typical local
things at home since years, and we used Odroid H2s to build an [18 node bare
metal lab cluster at
work](https://photos.google.com/share/AF1QipPIxF5isLFw8q3Y5bL6p22sNWmxLYC7JQUArTgIg4MjGRWVMu8LyGeXqT3R3Gx_gA?key=Z1ZZc3Z1bnAxakNpbEdfRTFLbk5TWDRBNXRUal93)
to build a test environment for things which cannot be tested in kind, minikube
and other virtual envs (like simulated blackout for a complete dc, a failing switch,
storage going crazy...).

### Switch / Router

To reduce the blast radius and protect my home network i searched for a little
router to separate the home network from the lab network.  I opted for a
[MikroTik hEX RB750Gr3](https://mikrotik.com/product/RB750Gr3). MikroTiks routerOS
allows a lot of different configurations if neccessary.  I use one port as
uplink into the home network.

My H2 is dual homed, one leg in the home network, the other in the lab network.
This leg should act as provisioner for the lab cluster.

### Cooling

My H2 has a pwm controlled 92mm fan on the top of its case, but it barely uses
it under normal work load.

The case (see below) is somehow size constrained, so i decided to use 80mm
[Noctua NF-A8 PWM](https://noctua.at/en/nf-a8-pwm) fans, suitable to the
standard 4pin 2,54mm pitch connector.

### Power Supply

The Odroids drain up to 4 Amps each with two SATA disks and full cpu load with
15V DC, the MikroTik is also fine with 15V, so i bought a [ham radio power
supply](https://www.komerci.de/shop/stromversorgung/Festspannungsnetzgeraete/ps30swiv-festspannungsnetzgeraet-13-8v-30a-lcd)
with 15V and 30 Amps max) and some 5.5 mm dc power jacks with open wires and
build a suiable power distribution cable.

In real live all loads together consume under 3A.

### Case 

I printed a case with parts of
[Odroid H2 Rackmount project](https://www.thingiverse.com/thing:3485530) - the H3 
have the same physical specs as their predecessor H2.

![Case with Odroids and Switch](pics/case_sm.jpg)

I needed to burn a lot of smaller filament remains, so it has really ugly colours :-)
[Here are some more pictures](https://photos.google.com/share/AF1QipOEYq0544IV67harl58_uC0024xNleLqJeiRTEjn7_saC3fTc6Ne1Pnuho2mmJ2EA?key=SUhpWUtIOFYzX0pybnV2RXV3aVNjRk9uWXVsazFR) 

This case design is genius while it is not only endless stackable but it
has also cages/bays and caddies, the Odroids are mounted to the caddies and
are easy removable from the rack. Thats not the case (sic!) at our company labcluster
case, it is really a mess to change a node, if neccessary.

The switch is just 1mm to big to fit into a cage, thats why it lives on the top
without the vertical columns.

I used 3mm rod to connect the units, the holes are for much bigger rod, so i
printed some spacers and pressed them into the connections between the units
([these blue tubes](https://photos.google.com/share/AF1QipOEYq0544IV67harl58_uC0024xNleLqJeiRTEjn7_saC3fTc6Ne1Pnuho2mmJ2EA/photo/AF1QipOLIP7ZdU2PIErlum0OlAI_0ENNHN7T6_IcpPRl?key=SUhpWUtIOFYzX0pybnV2RXV3aVNjRk9uWXVsazFR)).

The fan base is the longest thing i ever printed, it uses the whole printspace
my printer is able to serve (25cm). I melted threaded inserts into it to mount
the fans on it.

