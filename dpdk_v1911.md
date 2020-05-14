``
Upgrade DPDK library version to 19.11 LTS
-----------------------------------------
Introduction:
-------------
Contrail vrouter currently uses DPDK version 18.05.1 which is a little less
than 2 years old. Lot of enhancements, bug fixes, support for new NIC firmware
etc. has gone in after that.

Problem statement:
------------------
Contrail vRouter should be updated to support the latest DPDK 19.LTS. The DPDK
19 series brings support for:

- new intel NIC’s (PAC N3000) and firmware
- enhancements around container networking DPDK
- many bug fixes
- virtio library fixes (required for upstreaming contrail vhost infrastructure)
- live migration fixes
- fixes in bonding/lacp
- bug fixes for mellanox NICs
- multiqueue enhancements (to support more queues than the forwarding cores)

Requirements:
-------------
1) Contrail code should work for both DPDK 18.05.1 and 19.11(LTS) for some time
2) Contrail code should give same/more performance
3) Contrail code should support live migration
4) Contrail code should move to use upstream vhost library

Changes needed:
---------------
DPDK library changes -

1) eal: changes for setting control thread mask 
2) net/bonding: Support for LACP fast timers 
3) net/af_packet: changes to accomodate jumbo frames 
4) net/bonding: Changes for accomodating jumbo frames in bond
5) mbuf: Add add GSO flags 
6) config: Disable building DPDK kernel modules and increase max lcores to 256
7) net/bonding revert "fix LACP negotiation" 
8) Retain mbuf headroom as 128 since mirroring is now supported with
   any mbuf headroom

vrouter library changes -
-------------------------
1) Remove offloads flag from ethdev_conf. It should be used only in tx_queue_conf
2) Add rte_eth_allmulticast_enable() for bond bcast reception
3) Changes in API to get PCI address of bond - dpdk_find_port_id_by_pci_addr()
4) Enable VLAN offload
5) Remove vr_dpdk_knidev.c and its dependencies since its not longer supported
6) Some structure names like eth_header etc. have changed as rte_eth_header.
7) In ethdev and rx_queue offloads, remove flag DEV_RX_OFFLOAD_CRC_STRIP
8) Some cleanup in SConscript

UI changes -
------------
None

Schema changes -
----------------
None

Feature testing -
-----------------
TBD

Solution testing -
------------------
TBD

Upgrade impact -
----------------
None
```
