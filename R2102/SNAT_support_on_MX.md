# 1. Introduction
Contrail Fabric Manager (CFM) supports multiple physical roles and routing bridging (overlay) roles on QFX devices and MX devices currently. As part of the DC-Gateway overlay role, MX routers support the inline NAT functionality by using the PFE with FIP (Floating IP).

# 2. Problem statement
There is already VXLAN Public LR construct that is available in CEM that can be used to provide this functionality. When it's extended to MX DC Gateway, it should be aware that MX is equipped with an MS-MPC card and required configuration has to be pushed to MX to provide SNAT functionality. 
This SNAT should be available not only to BMS, but also to VMs, meaning that users will have a choice for VMs, what SNAT to use - local vRouter or DC-GW based. SNAT type LR will be left to serve its purpose in pure vRouter NS based SNAT scenario.

#### Supported Device Models
MX 240, MX 480, MX 960, MX 10003

# 3. Proposed solution

1. Is to make use of the existing public Logical Router.
2. New option will be asked to select the VN which needs to be used source address pool, while NAPT.
3. Is to make use of the service-interface property to ask the multi serivce interface, which will involve for this NAPT.

# 4. API schema changes

New LogicalRouterVirtualNetworkEnumType, NAPTSourcePool, introduced to support the virtual-network used as a source pool for NAPT

<xsd:simpleType name="LogicalRouterVirtualNetworkEnumType"><br/>
    <xsd:restriction base="xsd:string"><br/>
        <xsd:enumeration value="ExternalGateway"
            description='Virtual network is used as external gateway for
             this logical router. This link will cause a SNAT to be spawned
              between all networks connected to logical router and external
               network.'/><br/>
        <xsd:enumeration value="InternalVirtualNetwork"
            description='Virtual-network is used for EVPN Layer3 packets
             routing. This network is used to route packets from all
              children EVPN Layer2 networks linked this logical-router'/><br/>
        <xsd:enumeration value="NAPTSourcePool"
            description='Virtual-network is used as Source Pool for NAPT
             with Multi Service line cards (MS-MPC/MS-MIC) for MX Router
             '/><br/>
    </xsd:restriction><br/>
</xsd:simpleType><br/>

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
New dropdown needs to provided to select the Virtual Network as Source Poll when it's marked as Public Logical Router.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
As mentioend in the proposed solution, existing inline service NAT (PFE based) will be extented to support the MS-MPC based NAPT or SNAT.
Added new checks in the code to generate the required configuration based on the service interface name
i.e if the service interfaces
    starts with si-x/x/x it will be considered as PFE based Inline Service NAT.
    starts with ms-x/x/x it will be considered as Carrier-grade NAT.

Below CGNAT (NAPT) configuration will be pushed to MX router,

set groups __contrail_overlay_evpn_gateway__interfaces ms-2/0/0 unit 17 family inet
set groups __contrail_overlay_evpn_gateway__interfaces ms-2/0/0 unit 17 service-domain inside
set groups __contrail_overlay_evpn_gateway__interfaces ms-2/0/0 unit 18 family inet
set groups __contrail_overlay_evpn_gateway__interfaces ms-2/0/0 unit 18 service-domain outside
 
set groups __contrail_overlay_evpn_gateway__services service-set sv-_contrail_vn120-l3-9 nat-rules sv-_contrail_vn120-l3-9-sn-rule
set groups __contrail_overlay_evpn_gateway__services nat rule sv-_contrail_vn120-l3-9-sn-rule match-direction input
set groups __contrail_overlay_evpn_gateway__services nat rule sv-_contrail_vn120-l3-9-sn-rule term term_120_1_1_30 from source-address 120.1.1.0/24
set groups __contrail_overlay_evpn_gateway__services nat rule sv-_contrail_vn120-l3-9-sn-rule term term_120_1_1_30 then translated source-pool p1
set groups __contrail_overlay_evpn_gateway__services nat rule sv-_contrail_vn120-l3-9-sn-rule term term_120_1_1_30 then translated translation-type napt-44
 
set groups __contrail_overlay_evpn_gateway__services service-set sv-_contrail_vn120-l3-9 next-hop-service inside-service-interface ms-2/0/0.17
set groups __contrail_overlay_evpn_gateway__services service-set sv-_contrail_vn120-l3-9 next-hop-service outside-service-interface ms-2/0/0.18
  
set services __contrail_overlay_evpn_gateway__nat pool p1 address 70.0.0.0/24
set services __contrail_overlay_evpn_gateway__nat pool p1 port automatic random-allocation

# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Testing
## 14.1 Unit tests
One new UT case will be added to verify the source pool for DC-GW overlay role on MX Router.

## 14.2 Feature tests
Below is the link to the feature test plan,
https://drive.google.com/file/d/1E_Jd4v5IJx_iArJj5HFTWJKEZV9QQEeb/view?usp=sharing

# 15. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 16. References
https://contrail-jws.atlassian.net/browse/CEM-19169

# 17. Caveats or Known issues
There are no known issues.