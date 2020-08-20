# 1. Introduction
A collapsed spine architecture has no leaf layer. Instead, the Layer 3 IP-based underlay and the EVPN-VXLAN overlay functionality that normally runs on leaf switches is collapsed onto the spine switches. 
The spine switches also act as a border gateway.

A collapsed spine architecture with EVPN multihoming is ideal for organizations with:

    Plans to move to an IP fabric-based architecture with EVPN-VXLAN overlay.

    Small data centers with a mostly north-south traffic pattern.

    A need to extend Layer 2 traffic across data centers.

    Multi-vendor legacy ToR switches that do not support EVPN-VXLAN.

    Current or future requirements to support more than two spine switches to ensure adequate bandwidth during maintenance or a spine failure.

    A need for an alternative to an MC-LAG (ICCP protocol) architecture.

### Supported Deployment model:
Supported Roles:
- Collapsed-Spine@spine

### Supported Device Models:
    QFX5120-32C​, QFX5120-48Y-8C, QFX5120-48T-6C, QFX10002-72Q/36Q/60C, QFX10008/QFX10016​

### Supported Device version:
    18.4R2-S3 and above.

# 2. Problem statement
CFM should support the Collapsed Spine Topology, 2 to 4 spines.
No need to manage the L2 leaf switches.

# 3. Proposed solution
The propoese solution will introduce a new overlay role called "Collapsed-Spine" and leverage the existing architecture to configure the Virtual Networks, Logical Router and the Virtual Port Group to support this Collapsed-Spine topology.

Collpased-Spine functionality is support for both Brownfield and Greenfield scnearios.

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
     It will create a new overlay role called "Collapsed-Spine" and add it it to qfx10k node profile.
     Collapsed-Spine functionality is close match of the exisiting CRB-Gateway (existing role) for the Logical Router and we need to create a VPG.
     Collapsed-Spine role can be combined with DC-GW, DCI-GW, CRB-MCAST-GW roles to support as boarder gateway device.

     Below configuration is pushed with the new template in addition to the existing (CRB-GW) configurations.
    
     -overlay_evpn_collapsed_gateway

        set groups {{cfg_group}} protocols evpn no-core-isolation

        set groups {{cfg_group}} protocols evpn default-gateway do-not-advertise

        set groups {{cfg_group}} forwarding-options vxlan-routing next-hop 32768

        set groups {{cfg_group}} forwarding-options vxlan-routing overlay-ecmp

# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing

## 15.1 Unit tests
Minimum 3 UT cases, below will be added,
1. New role in the rb_roles of the abstarct configuration.
2. Ports details which used to connect the L2 leaf switches.
3. Check the feature configs generated for the Collapsed-Spine.

## 15.2 Feature tests
Below is the link to the feature test plan,
https://drive.google.com/file/d/1t-SR-WRdzrNvW-7wKAPzVa0TvcLescYe/view?usp=sharing

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# Known issues and Caveat
There are no known issues for this functionality.

# 18. References
https://contrail-jws.atlassian.net/browse/CEM-12963
https://www.juniper.net/documentation/en_US/release-independent/nce/topics/example/nce-178-collapsed-spine-configuration-example.html
