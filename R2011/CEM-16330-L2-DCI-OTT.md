1. Introduction
This blueprint describes the requirements, design and implementation of Layer 2 (L2) Data Center Interconnect (DCI) Over The Top (OTT) functionality support in contrail fabric.

2. Problem statement
Currently, there is no way user can stretch L2 virtual networks in contrail DCI functionality. Contrail DCI supports L3 overlay ebgp and ibgp across the fabrics through fabric's logical routers IRB. 

3. Proposed solution
User should have an option to configure DCI for L2 mode to leake the tenant virtual network's route targets across the fabric. User should allow to select the list of fabrics and list of tenant's virtual networks whose route targets will be extended across the fabrics through L2 DCI configuration. 
 
Contrail DCI will internally create overlay eBGP or iBGP session between the fabric devices whose RB Role has DCI-gateway. The eBGP session will be use between devices whose ASN are different else iBGP session will be use. This session will import policy defining community as a RT (Route Target) for L2 and L3 DCI configuration. Optionally user can create L3 mode DCI interconnect object for the same fabric, this will end up using the same overlay eBGP or iBGP session between the fabric devices to stretch both L2 and L3 policies between the fabrics. For L3 DCI configuration, session will import policy from community of peer farbic Logical Router's IRB route targets. For L2 DCI configuration, session will import policy from community of tenant virtual network's route target.

L2 DCI OTT will have following restriction:
- Current L2-DCI feature will be supported for QFX platform. MX Junos platform will be supported in future release.
- L2 DCI can be setup between the multiple fabrics in the same controller.
- If overlay DCI is needed for a fabric connected outside of the controller, the user will need to coordinate route targets and ASN.
- For L2 DCI Underlay eBGP configuration, user will use contrail master LR and routed Virtual Network ebgp configuration to achieve L2 DCI underlay eBGP peering between the fabrics. This document addresses L2 DCI Overlay eBGP and iBGP peering through new UX workflow.
 
4. User workflow impact
4-1) Contrail UI screen, Overlay->DCI->Data Center Interconnect->Create will have following configuration options for user to enter.
       - DCI Name
       - DCI Mode. This DCI Mode will have radio button to choose either L2 Or L3 mode. 
       - If selected DCI Mode L2, then following option will be provided to the user:
            - Fabrics. User must select minimum two or more fabrics. UI will display only those fabrics in this selection, who has one or more physical router device(s) with DCI-gateway RB role.
            - Available Virtual Networks - All Tenant created Virtual network excluding Routed Virtual network will be displayed here.
             -Selected Virtual Networks - User will select one or mode Virtual networks from the Available Networks.

4-2) Contrail UI screen, Overlay->DCI->Data Center Interconnect->Edit will have following configuration options for user.
       - DCI Name
       - DCI Mode. This DCI mode option will be disabled for user to select in Edit Screen. Contrail will not allow to modify existing DCI object's DCI mode.  
       - All other option will be the same as create screen where user will allow to add or remove fabric and tenant virtual networks in Edit Screen.
       
4-3) Contrail UI screen, Overlay->DCI->Data Center Interconnect will display the list of existing DCI objects with following details
       - DCI Name
       - DCI Mode
       - Connnection.
     On clicking one of the DCI name object who has L2 DCI mode, UI will display details of selected DCI object's Fabrics and Virtual network list.

UI and DCI object Validation:
Following Validation will be done in UI and in back-end Contrail API Server code, both places to make sure only the valid L2 mode DCI object will get saved at create or edit operation.
    - L2 DCI mode object must have minimum two fabric and one virtual network.
    - Only those Fabric will be allowed to be selected in L2 DCI mode who has at least one DCI-Gateway RB Role assigned to Physical Router Device.
    - Only non routed type virtual network will be allowed to be selected in L2 DCI mode.  

5. API Schema changes
In data-center-interconnect object, following New type value l2 and fabric refs property will be added. 

<xsd:simpleType name="DataCenterInterconnectModes" default='l3'>
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="l3"/>
         <xsd:enumeration value="l2"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:element name="data-center-interconnect-fabric"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('data-center-interconnect-fabric',
             'data-center-interconnect', 'fabric', ['ref'], 'optional', 'CRUD',
             'Reference to fabric, this link enables to identify which fabric this DCI belongs to. This refs used for l2 mode inter-fabric dci') -->

6. Performance and scaling impact
None

7. Deprecations
None

9. Implementation
DM module uses abstract config to pass configuration to jinja files which gets push to the juniper device. Following new two properties 'import-policy' and 'policies' will be added to Bgp abstract configuration.

<xsd:complexType name="Bgp">
    <xsd:all>
       <xsd:element name="comment" type="xsd:string"/>
       <xsd:element name="name" type="xsd:string"/>
       <xsd:element name="uuid" type="xsd:string"/>
       <xsd:element name="type" type="xsd:string"/>
       <xsd:element name="autonomous-system" type="xsd:integer"/>
       <xsd:element name="cluster-id" type="xsd:integer"/>
       <xsd:element name="ip-address" type="xsd:string"/>
       <xsd:element name="hold-time" type="xsd:integer"/>
       <xsd:element name="authentication-key" type="xsd:string"/>
       <xsd:element name="authentication-method" type="xsd:string"/>
       <xsd:element name="families" type="xsd:string" maxOccurs="unbounded"/>
       <xsd:element name="peers" type="dac:Bgp" maxOccurs="unbounded"/>
       <xsd:element name="bfd" type="dac:Bfd"/>
       <xsd:element name="multihop" type="dac:MultiHop"/>
       <xsd:element name="routing-policies" type="dac:RoutingPolicyParameters"/>
       <xsd:element name="import-policy" type="xsd:string" maxOccurs="unbounded"/>
       <xsd:element name="policies" type="dac:PolicyParameters" maxOccurs="unbounded"/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="PolicyParameters">
    <xsd:all>
        <xsd:element name="name" type="xsd:string"/>
        <xsd:element name="comment" type="xsd:string"/>
        <xsd:element name="import-targets" type="xsd:string" maxOccurs="unbounded"/>
    </xsd:all>
</xsd:complexType>

DM module will generate DCI BGP abstract config having common policies between the list of peers. This policies will be imported in bgp group session. The policy will define community of route targets allow to be imported in BGP session. 

9. Testing
https://docs.google.com/spreadsheets/d/1MU61n8LrGYy9FAya9E0D6wSGvFiTD3_P2dbdQQZ_xNo/edit?usp=sharing

10. Documentation Impact
It will be documented as part of release documentation by the doc team.

11. References
https://contrail-jws.atlassian.net/browse/CEM-16330
