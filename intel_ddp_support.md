Support for Intel DDP
=====================


Introduction
------------
This story is part of the performance initiative that is undertaken from the
beginning of 2020 to improve the performance of DPDK vRouter.

Background
----------
Contrail dataplane runs on general purpose x86 servers called compute nodes.
In contrail, all the overlay MPLS or VXLAN tunnels terminate at the compute
node instead of hardware switch like QFX. They terminate at a software switch-
cum-router called vRouter which is running in all the computes and it is the
main dataplane component of contrail. vRouter can either run as a kernel module
or a DPDK application. The most common use-case is DPDK mode. It is the
workhorse of the contrail system. Each and every packet in the contrail system
flows through vRouter. It essentially forwards packets between VMs and physical
interface. It is a multi-core, multi-threaded component since it needs to be
highly performant.

Packet process in DPDK vRouter
------------------------------
Step 1 - Packets arrive at the physical interface or NIC card of the server and
the NIC hardware load balances them and distributes them into different interna 
hardware queues based on the 5-tuple hash of the packets.

Step 2 - For each hardware queue which is configured, the vRouter allocates a
dedicated CPU core to poll the packets and dequeue it for further processing.

Step 3 - For certain types of packets, the NIC card cannot load balance the
packets properly. In vRouter software, we can figure this out. If NIC has
load balanced it properly, the same CPU core which dequeued the packet will do
the packet processing also. If not, the receiving CPU core performs software
load-balancing by distributing the packets to other CPU cores which will do the
processing.

Step 4 - The final step is the CPU cores, after processing, puts them into one
of the virtual machine Tx queues from where the VM gets the packets

Problem statement
-----------------
Packets having the encapsulation type as MPLSoGRE does not have enough entropy.
Entropy means it does not have enough information for load-balancing. In
specific, they lack port information. Compare it with MPLSoUDP where we have the
full 5 tuples Source IP, Destination IP, Source Port, Destination Port, Protocol.
But in MPLSoGRE, we only have Source IP, Destination IP and protocol. So NIC
cannot properly load balance the packets. All packets between a pair of computes
land up in the same Rx Queue of the NIC. Due to this, the CPU core which polls
that Rx queue becomes bottleneck for the whole compute and the performance
suffers.

Solution
--------
We introduce a new capability to contrail dataplane which removes the bottleneck
for MPLSoGRE packets so that performance is proportional to the number of CPU
cores. That is, no single CPU core can be a bottleneck and packets are
distributed evenly across different hardware Rx queues. We use Intel dynamic
device personalization technology or DDP in short to achieve this. With Intel
700 series, the fortville family, Intel moved to programmable pipeline model
from a fixed pipeline model which existed in the earlier Niantic family. They
added this new capability. DDP allows dynamic re-configuration of packet
processing pipeline in the NIC at runtime without server reboot. Software can
apply custom profiles to the NIC so that it can classify new packet types inline,
and distribute these packets to specified Rx queues

Parsed fields before enabling NIC with DDP
------------------------------------------
DA | SA | IPv4 | GRE | MPLS | DA | SA | IPv4 | UDP | PAYLOAD
\____________________/

In the above, MPLSoGRE is unknown flow type, so no RSS and other filters
possible. The NIC is able to parse only till the GRE header where in it does
not get the port information which is present in the UDP header present in the
inner packet. So it does not have enough information to distribute the packets
properly. RSS is nothing but the way in which the NIC determines the hash and
put it into the corresponding hardware queue.

Parsed fields after enabling NIC with DDP
-----------------------------------------
DA | SA | IPv4 | GRE | MPLS | DA | SA | IPv4 | UDP | PAYLOAD
\__________________________________________________/

In the above, MPLSoGRE is known flow type, so all filters like RSS possible
based on inner IP and inner UDP headers. The NIC can start recognizing the
inner packet header including the inner IPv4 and inner UDP headers and the
entropy increases because it can start using the UDP port information to compute
the RSS hash.

DDP toolchain
-------------
For DDP, the end-user need to create a custom profile for their packet type.
This can be created by using a tool-chain from Intel, called profile editor.
Intel has published some standard profiles too which can be directly downloaded
from intel website and used.

Step 1 - This profile editor can be used to create new entries or modify
existing entries for the NIC analyzer or parser

Step 2 - Create a new profile for MPLSoGRE packets which defines the structure
of packet headers for this encapsulation

Step 3 - Compile and create a binary package (.pkgo) which can be applied to
the NIC

Step 4 - Now the NIC is enabled to recognize MPLSoGRE packets

Code changes
------------
DDP feature applicable only for forville NIC family. When DPDK bootup with
command-line argument as --ddp and NIC driver as "net_i40e", vrouter programs
DDP profile stored in "/var/lib/ddp/mplsogreudp.pkg" for all ports.

Suppose, contrail-vrouter-dpdk passed with command line argument as --ddp and
NIC is not fortville(X710 series), vrouter silently ignores loading DDP profile
and bootup asusual.

Performance measurements
------------------------
MPPS   | Without DDP | With DDP | % Gain
1 core | 2           | 2        | 0.00
2 core | 3.8         | 3.8      | 0.00
3 core | 5.61        | 5.61     | 0.00
4 core | 6.48        | 7.48     | 15.43
6 core | 6.58        | 11.22    | 73.15

Latency (us) | Without DDP | With DDP | % Gain
Avg Latency  | 311         | 181      | 41.80
Max Latency  | 4154        | 768      | 81.51

Deployments supported
----------------------
- RHOSP
- Juju
- Kolla

CLI changes
-----------
dpdkinfo tool will display the status of DDP
New CLI added dpdkconf to add/delete DDP profile image.
dpdkinfo --ddp list -> To display the DDP profile.
dpdkconf --ddp add -> Add DDP profile, if NIC is fortville.
dpdkconf --ddp delete -> Delete already loaded DDP profile.

Default configuration
---------------------
By default DDP is not enabled, use the new command-line option --ddp to enable
DDP for fortville NIC.

Command-line options
--------------------
New command-line option --ddp to enable DDP only for fortville NIC family.

UI changes
-----------
None

Upgrade changes
----------------
None

Test cases
-----------
TBD
