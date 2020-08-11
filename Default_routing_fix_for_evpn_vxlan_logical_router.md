
# 1. Introduction

A logical router is a  component that forwards data packets between networks. It also provides Layer 3 and NAT forwarding to provide external network access for virtual machines on project networks. Logical router in contrail uses evpn for control plane programming and vxlan for forwarding.

# 2. Problem statement

The current LR implementation addresses the E-W L3 VM-VM, BMS-BMS and VM-BMS traffic flows between interconnected VN’s. However when there is a 0.0.0.0/0 route coming from MX or when the tenant VN is extended to a SNAT-GW this breaks the routing topology due to absence of specific routes injected into the tenant VRF by agent.

# 3. Proposed solution

1]. Inter-subnet E-W routing between the Contrail VNs should be done by agent injecting the non-/32 VN subnet route for the IPAM subnet of all the other remote-VNs that are connected to the LR. This way there is no need for 0.0.0.0/0 to be added to the tenant VRF by the agent.

2]. In order to provide connectivity to the Customer/PNF subnets QFX10K advertises these routes to Control-node which is installed in the LR1 EVPN T5 VRF as BGP routes. Agent should install these routes in all the connected tenant VRFs so that the tenant VRFs can reach the PNF/Customer subnets using QFX10K VTEP next-hop.

3]. For the N-S internet connectivity through MX the BGP vpnv4 route (0.0.0.0/0) advertised by MX should be used for this.

4]. It is also possible that QFX10K itself can advertise 0.0.0.0/0 ( because it is getting the 0/0 from PNF or Public=True is set in LR config) in which case 0.0.0.0/0 should be added to the tenant VN by the agent. Both MX and QFX10K advertising 0/0 is not recommended, however if this happens to be the case 0/0 to MX will be preferred (BGP over EVPN).

5]. Instead of 0.0.0.0/0 directly coming from the MX it is possible that the tenant VN is connected to a SNAT LR. In this case the 0.0.0.0/0 route in the tenant VN-VRF points to the SNAT LR (instead of MX) and for inter-subnet VN-VN traffic or traffic destined to PNF should follow the requirements #1 and #2.

## 3.1 Alternatives considered

None

## 3.2 API schema changes

None

## 3.3 User workflow impact

None

## 3.4 UI changes

None

## 3.5 Notification impact

None

# 4. Implementation

## 4.1 vRouter Agent

1]. If there are ‘n’ networks connected to a VxLAN LR, insert subnet routes of n-1 networks into each native VRF.
2]. If any non /32 route ( that could be a dynamic EVPN T5 BGP route coming from QFX including 0/0) gets added to LR VRF, it will be copied to all connected VRFs. This will require agent to listen on LR VRF table.
3]. If a VN is connected to both SNAT and VxLAN LRs and the VxLAN LR also has default route, there will be 2 paths to the def.route.
In this case, the SNAT-LR path will be preferred as BGP Paths have a higher preference compared to EVPN-ROUTING paths(Added by agent).

# 5. Performance and scaling impact

1]. There is no impact on any of the Contrail object scaling.

2]. The only attribute that can impact the validated scale is the # of customer routes received from the 3rd party PNF via QFX10K. The recommended scale for the external 3rd party routes received from QFX border gateway is average=25 routes/LR and maximum=100 routes/LR. This scale is applicable for all the EVPN T5 LRs that is created in the system. Verify 100 default routes received from gateway in LR vrf as part of scale test.

## 5.1 API and control plane

No impact.

## 5.2 Forwarding performance

No impact.

# 6. Upgrade

None.

# 7. Deprecations

None.

# 8. Dependencies

None.

# 9. Testing

## 9.1 Unit test
1.  Create one port each in vn1 and vn2 both having 2 ipams, connect vn1 and vn2 to a logical router.
    Verify that both subnet route for vn2 is present in vn1 vrf, also verify vn1 subnet route present
    in vn2 vrf.
2.  Add a default route in LR vrf from above topplogy and verify it gets added to vn1 and vn2 vrf.
    Add a non /32 route in LR vrf and verify it gets added in vn1 and vn2 vrf.
3.  Delete a subnet from vn1 and verify subnet route gets deleted from vn2 vrf.
4.  Delete a non /32 route from LR vrf and verify it gets deleted in vn1 and vn2 vrf. 
5.  Create one port in vn1 and vn2 each and add a Snat router to vn1 and vn2 also add
    vxlan LR with default route to vn1 and vn2 , verify path with evpn peer is has higher priority.

## 9.2 Feature test

Test Plan: https://drive.google.com/file/d/1rmHdcfmC5RK9lDKEx5pyd-JeSJE0YBdr/view?usp=sharing

## 9.4 Solution test

1]. Verify 100 default routes received from gateway in LR vrf as part of scale test.
[Test team to add more]

# 10. Documentation Impact

1]. If a VN is connected to both SNAT and VxLAN LRs and the VxLAN LR also has default route, there will be 2 paths to the def.route.
In this case, the SNAT-LR path will be preferred as BGP Paths have a higher preference compared to EVPN-ROUTING paths(Added by agent).
In this case VxLAN LR default route will not be used, this is expected but needs to be documented. Gateway for SNAT in this case can
be set accordingly to address unused default route (reachability) from VxLAN LR.

# 11. References

1. Default route fix for LR story: https://contrail-jws.atlassian.net/browse/CEM-15909

