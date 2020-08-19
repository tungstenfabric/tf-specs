# 1. Introduction
The edge-routed bridging overlay performs routing at IRB interfaces located at the edge of the overlay (most often at the leaf devices). 
As a result, Ethernet bridging and IP routing happen as close to the end systems as possible, but still support Ethernet dependent applications at the end system level.

# 2. Problem statement
In ERB Fabric when creating a LR via CEM (and attaching VNs), CEM is creating the IRB for each associated-VNs on all the ERB switches disregard if there is a local port (VPG) in the associated-VNs.

This is very harmful because it reduced the scale of the Switch and the Fabric (and introducing new challenges: large configuration, convergence is affected in some cases, more BUM crossing the Fabric,…).

CEM-managed Fabric will configure on each ERB Node:

    All Routing-Instances

    All IRB

# 3. Proposed solution
As per the current design we are pushing the VLAN, VNI and IRBs related to all the Virtual Networks part of the LR, which extended to ERB leaf. 

Use the VPG (Virtual Port Group) to configure the resources like, IRB (Integrated Routing and Bridging), VNI (VXLAN network identifier), VLAN (Virtual LAN) on the switches.

This is applicable to both MX ad QFX families.

# 4. API schema changes
N/A

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
N/A

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
As per the new ask we need to configured the menteiond resources if we have any VPG created on the leaf for the respective VN.

If there's no VPG/VMI for a VN which is part of LR then it should not be configre the VLAN, VNI and IRB related to respective VN in the leaf.

Topology:

    In the below topology one spine is connected to 2 leafs and every leaf is connected with a BMS.

    1. BMS1 (VN90) --> leaf1 (ERB-UCAST-GW) --> spine1 (lean, RR) -- leaf2 (ERB-UCAST-GW) -- BMS2 (VN91)

    2. Create LR with VN90, VN91 and extened to leaf1 and leaf2.

    3. Create a VPG1 with BMS1 to leaf1 with VN90 and vlan-id 90.
       Create a VPG2 with BMS2 to leaf2 with VN91 and vlan-id 91.

    Existing config push on leaf1 and leaf2:

        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 10 family inet address 91.91.91.1/24
        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 10 mac 00:00:5e:01:00:01

        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 9 family inet address 90.90.90.1/24
        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 9 mac 00:00:5e:01:00:01

        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 description "Virtual Network - vn_91"
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 l3-interface irb.10
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 vlan-id 91
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 vxlan vni 10

        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 description "Virtual Network - vn_90"
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 l3-interface irb.9
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 vlan-id 90
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 vxlan vni 9

    With the new change:

    Leaf1 is having a VPG with VN90 (BMS1) config to leaf1:
        
        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 9 family inet address 90.90.90.1/24
        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 9 mac 00:00:5e:01:00:01

        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 description "Virtual Network - vn_90"
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 l3-interface irb.9
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 vlan-id 90
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-9 vxlan vni 9

    Leaf2 is having a VPG with VN91 (BMS2) so the config to leaf2:

        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 10 family inet address 91.91.91.1/24
        set groups __contrail_overlay_evpn_ucast_gateway__ interfaces irb unit 10 mac 00:00:5e:01:00:01

        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 description "Virtual Network - vn_91"
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 l3-interface irb.10
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 vlan-id 91
        set groups __contrail_overlay_evpn_ucast_gateway__ vlans bd-10 vxlan vni 10

# 11. Performance and scaling impact
This will introduce some new loops to check the VPGs on the leafs on the ERB topology with ERB-UCAST-GW role, we need to run scale TCs to see the impact.

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing
Configuration on the ERB leaf will be defined by the VPGs with associated VNs, so we need to add minimum below UT caes

## 15.1 Unit tests
Minimum 3 UT cases, below will be added,
1. Check all the VLAN, VNI and IRBs created at every leaf for the all VPGs.
2. There should not be any VLAN, VNI and IRBs for ones where we don't ahve any VPG connected.
3. Add a new VN to the VPG and see whether the corresponding VNI, VLAN, IRB configured or not.

## 15.2 Feature tests
Below is the link to the feature test plan,
https://drive.google.com/file/d/1CNmLM300_oi7bMq2nRM05FUop3MQzHIf/view?usp=sharing

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# Known issues and Caveat
There are no known issues for this functionality.

# 18. References
https://contrail-jws.atlassian.net/browse/CEM-15800
https://www.juniper.net/documentation/en_US/release-independent/solutions/topics/task/configuration/edge-routed-overlay-cloud-dc-configuring.html
