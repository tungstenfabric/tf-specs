```
Latency Improvements in vRouter
-------------------------------

Introduction:
-------------
Latency is the amount of time taken by the forwarding software to process the
packet. This is one of the key metric in determining the performance of any
dataplane. It is desirable to have as low latency as possible for any
application. Latency is introduced in the software due to the presence of 
internal queueing and also the actual processing like table lookups, header
manipulation, adding/deleting headers, re-writing some of the header fields etc.
For some applications like voip, 5G etc. the latency numbers are very crucial.
They are not latency and jitter tolerant. So it is important to always keep
the latency numbers as low as possible.


Contrail vRouter has many internal queueus. There are queues to the VNF, queues
to the physical interface and queues between the different forwarding cores.
All of these queueing stages introduce latency. Also, the bigger the size of the
queues, the larger is the latency. The trade-off with smaller queue size is that
the software will not be tolerant to traffic bursts. So there needs to be a
balance between the queue size and burst threshold

Background:
-----------
Before R2003 release, the latency numbers for 64B packets was in the order of
300 - 400 us. This number went to around 800 us with the performance improve-
ments we did in R2003 release where we introduced a CLI knob to control the
size of the queues between lcores. The measurements we did was with size of
2K (default was 1K). Due to this bigger queue size, the latency numbers also
went up when we tested vRouter for maxmimum PPS. But with some more experiments
we did, we determined that latency comes down sharply if we target for a
reduced PPS. Ideally, we need to design the software such that latency numbers
are independent of the queues and queue sizes.

Pipeling vs Run-to-completion model:
------------------------------------
Pipeling model is a model where a software is divided into multiple stages. Each
stage completes part of the processing and hands it over to the next stage and
so on. The way to handover is through the use of a FIFO between the stages.
These FIFOs are potential latency killers. But the trade off is that the
architecture is more robust to ensure more load balancing and more cache
friendly.

Run-to-completion model is a model where the software does not have multiple
stages and it does the entire processing in a single context or single stage.
There are no FIFOs here ensuring that latency overheads are less. But this may
not be very cache friendly, specifically I-cache. Also, more constraints on
load balancing needs to be imposed.

In DPDK mode, our contrail vRouter uses a hybrid mode where it uses part pipe-
lining and part run-to-completion there by ensuring good load balancing and
also a reasonable latency. But is has a dependency on the FIFOs in the system.

Problem statement:
------------------
The latency numbers of DPDK vRouter are on the higher side. A typical 5G
deployment needs around 150us of latency (Needs verification). Our vRouter,
with default queue sizes (without the CLI parameters - vr_dpdk_rx_ring_sz and 
vr_dpdk_tx_ring_sz) gives around 300-400us of average latency. For maximum
PPS, we tune those CLI parameters to 2K size. With this setting, the latency
number shoots to 800us. This is highly unacceptable for these VNFs and needs
tuning.

Solution:
---------
For latency sensitive profiles, we need a way for vRouter to switch from hybrid
model to run-to-completion model. This means, the packets originating from the
VNF needs to be dequeued, processed and sent out to the wire, all by a single
forwarding core. Same should be the case in the other direction. Currently this
happens in the other direction for MPLSoUDP/VxLAN packets.

Current methodology:
--------------------
1) Packets originating from VNF to vRouter: The polling lcore dequeues the
packet, then computes the 5-tuple hash on the packet header, then sends it to
a different lcore which performs vRouter processing and sends out to the wire.

2) Packets entering vRouter from wire: The polling lcore dequeues the packet,
then checks the RSS and encap. If the encap is MPLSoGRE, it sends it to a
different lcore which performs vRouter processing and sends out to the wire.

New methodology:
----------------
In the new methodology, we introduce a knob "--vr_no_load_balance". If this
knob is set, no load balancing happens in (1) or (2) above. Basically, the
core which dequeues the packet also performs packet processing and sends it
out on the destination interface. Also, RSS hash computation in software is
disabled.

Limitations:
------------
To effectively use this feature, multiqueue virtio needs to be enabled so that
VNF itself distributes the load on the queues. Each queue is polled by different
lcores which avoids the use of lcore-to-lcore queueing. Without multiqueue, the
performance degradation will be significant since the whole packet processing 
for the VM interface will be limited to just one core.

For MPLSoGRE packets, the NIC cannot effectively load balance since there is
not much entropy in the packets. To overcome this limitation, Intel DDP needs
to be enabled with supported NIC cards.

Deployments supported:
----------------------
- RHOSP
- Juju
- Kolla

UI changes:
-----------
None

Upgrade changes:
----------------
None

Test cases:
-----------
TBD

Results:
--------
Latency Before: 700us
Latency After:  150us

```
