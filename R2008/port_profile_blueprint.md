# 1. Introduction
Port profiles are applicable to both L2 and routed VPGs.
This feature is about adding certain properties to port
profiles in consideration with some of the junos best practices.
These port profiles can be then attached to a VPG in order to
provide the desired characteristics.

Port profiles at present only consist of storm control profiles
as subtending objects. We include the following port attributes to port
profile object as part of R2008.

1. Port Attributes
       a. Port MTU
       b. Port Description
       c. Port Enable/Disable
2. Flow Control
3. LACP Attributes
       a. Enable/Disable
       b. LACP interval
       c. LACP mode
       d. LACP force-up
4. BPDU Loop Protection
5. Quality of Service
       a. Untrust Interface

# 2. Problem statement
Each interface characteristics and requirements on each device may be different.
This requires ability to specify custom configuration requirements and properties
for different interfaces on same or different devices. Port profile objects are
designed to target custom config push on devices and interfaces pertaining to user
requirements.

# 3. Proposed solution
After creating a port profile, it can be assigned to interfaces on one
or more switches. These port profiles can then be attached to a VPG for
pushing custom configuration specified by the port profile into the device.

With options to add more port profile attributes, the scope of creating
and applying different configurations to interfaces is enhanced.

The following are the accepted values for some of the port profile parameters:
1. Port MTU (bytes): Accepted Range: 256-9216 (in bytes)

Also note the following rules regarding the port profile attributes:

1. Since LACP force-up is recommended on a single individual member
   of an aggregate link, it is applicable only at the physical interface
   level and not at the VPG level. Therefore, this attribute is not
   applied as one of the port profile property.
2. LACP force-up option is applicable only to member links of VPG. In other words,
   it is applicable only to 'access' type interfaces.
3. LACP force-up option is not applicable on MX.
4. Few of the properties are applicable not only at the VPG level (by
   attaching port profile to VPG) but also at physical or logical interface
   level.
5. Following are the properties applicable at the physical interface level
       a. Port Attributes - MTU, Description, Enable/Disable. However
          Junos does not allow MTU configuration on individual physical
          interfaces (member links) of a VPG.
       b. Flow Control
       c. LACP force-up
       d. Physical Interface Type - configurable only if not pre-configured
          from backend.
6. Following are the properties applicable at the logical interface level
       a. Port Attributes - MTU, Description, Enable/Disable
       b. Quality of Service
7. Quality of Service is not extensively implemented for R2008. At present it
   just consists of applying a classifier based on 1P bits to all ethernet-switching ports.
8. Flow Control option is not applicable for 10K switches.

# 4. API schema changes

Objects to represent the port profile properties and interface
properties are added to the schema. The schema changes are captured below:

<xsd:complexType name="QualityOfServiceParams">
    <xsd:all>
        <xsd:element name="untrust-interface" type="xsd:boolean" default="false"/>
    </xsd:all>
</xsd:complexType>

<xsd:simpleType name="LacpMode">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="active"/>
         <xsd:enumeration value="passive"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name="LacpInterval">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="fast"/>
         <xsd:enumeration value="slow"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="LacpParams">
    <xsd:all>
        <xsd:element name="lacp-enable" type="xsd:boolean" default="false"
            description="Enable or disable this option for configuring LACP related params" />
        <xsd:element name="lacp-interval" type="LacpInterval"
            description= "Timer interval for periodic transmission of LACP packets. Fast mode receives packets every second. Slow mode receives packets every 30 seconds." />
       <xsd:element name="lacp-mode" type="LacpMode"
            description= "Configure this mode to active to initiate transmission of LACP packets and respond to LACP packets. LACP packets are not exchanged with passive mode." />
    </xsd:all>
</xsd:complexType>

<xsd:simpleType name="PortMtu">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="256"/>
        <xsd:maxInclusive value="9216"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="PortParameters">
    <xsd:all>
        <xsd:element name="port-disable" type="xsd:boolean" default="false"/>
        <xsd:element name="port-mtu" type="PortMtu" />
        <xsd:element name="port-description" type="xsd:string" />
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="PortProfileParameters">
    <xsd:all>
       <xsd:element name="port-params" type="PortParameters"
            description= "User can select this option to configure port parameters such as description, MTU and port enable or disable." />
       <xsd:element name="flow-control" type="xsd:boolean" default="false"
            description= "User can enable this option to configure flow control." />
       <xsd:element name="lacp-params" type="LacpParams"
            description= "Represents LACP configuration parameters." />
       <xsd:element name="bpdu-loop-protection" type="xsd:boolean" default="false"
            description= "User can enable this option to prevent loops on edge interfaces. This is applied on unit with family ethernet-switching." />
       <xsd:element name="qos-params" type="QualityOfServiceParams"
            description= "This is to configure any parameters affecting quality of service" />
    </xsd:all>
</xsd:complexType>

<xsd:element name="physical-interface-flow-control" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('physical-interface-flow-control', 'physical-interface', 'optional', 'CRUD',
              'User can enable this option to configure flow control on the physical interface.') -->

<xsd:element name="physical-interface-lacp-force-up" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('physical-interface-lacp-force-up', 'physical-interface', 'optional', 'CRUD',
              'User can enable this option to make LACP up and running. This is applicable only on individual member physical interfaces of a VPG.') -->

<xsd:element name="physical-interface-port-params" type="PortParameters"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('physical-interface-port-params', 'physical-interface', 'optional', 'CRUD',
              'User can select this option to configure port parameters such as description, MTU and port enable or disable.') -->

<xsd:element name="logical-interface-port-params" type="PortParameters"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('logical-interface-port-params', 'logical-interface', 'optional', 'CRUD',
              'User can select this option to configure port parameters such as description, MTU and port enable or disable.') -->

<xsd:element name="logical-interface-qos-params" type="QualityOfServiceParams"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('logical-interface-qos-params', 'logical-interface', 'optional', 'CRUD',
              'This is to configure any parameters affecting quality of service.') -->

<xsd:element name="port-profile-params" type="PortProfileParameters"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('port-profile-params', 'port-profile', 'optional', 'CRUD',
              'This is to configure port attributes.') -->

# 5. Alternatives considered
Apart from the solution discussed above, the following option was considered:
Option1: Provide other non-port related options at the VPG level. However
         since these options could be encapsulated into a port profile,
         there will be one place where user could specify the requirements
         and simply attach the port profile to the VPG while creation.

# 6. UI changes / User workflow impact

#### - Select Port Profiles from the overlay features
User selects the 'Port Profiles' option from the list of overlay features
![Overlay features list](images/storm_control_overlay_feature_list.png)

#### - Select Port Profile Attributes and storm control profile
User selects various Port Profile Parameters

#### - Port profile attachment to VPG
The port profile can be assigned to the VPG here. The port profiles
drop down will display the existing port profiles.
![Storm control VPG assignment page](images/storm_control_vpg.png)

#### - Edit Physical Interface
User sets various physical interface properties

#### - Edit Logical Interface
User sets various logical interface properties

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
The main implementation changes include:

1) Device manager changes to cache these new interfaces and port
   profile object properties.

2) Device manager changes to generate abstract config changes upon creation,
updation and deletion of the port profile objects and interfaces.

3) Jinja templates to generate the port profile configuration based on
   user input.

The configurations for QFX and MX vary slightly and the sample configs
are captured below:

#### Applicable for QFX10K, QFX5K and MX

##### 1. Port Description
set interfaces ae1 description "AE link"
set interfaces xe-0/0/0 description "AE link"
set interfaces <*> unit 0 description

##### 2. Port MTU
set interfaces et-0/0/48 mtu <256-9216>

##### 3. Port Enable/Disable
set interfaces ae1 disable
delete interfaces ae1 disable
set interfaces xe-0/0/0 disable
delete interfaces xe-0/0/0 disable

##### 4. LACP Enable/Disable
set interfaces ae0 aggregated-ether-options lacp
delete interfaces ae0 aggregated-ether-options lacp

##### 5. LACP Interval
set interfaces ae0 aggregated-ether-options lacp periodic fast
set interfaces ae0 aggregated-ether-options lacp periodic slow

##### 6. LACP Mode
set interfaces ae1 aggregated-ether-options lacp active
set interfaces ae1 aggregated-ether-options lacp passive

##### 7. BPDU Loop Protection
set protocols rstp bpdu-block-on-edge interface ae0

##### 8. Quality of Service
set class-of-service classifiers ieee-802.1 IP-UNTRUST
forwarding-class best-effort loss-priority
low code-points [000 001 010 011 100 101 110 111]

set class-of-service interfaces ae0 unit 0 classifiers ieee-802.1 IP-UNTRUST

#### Applicable only for QFX - LACP force-up
set interfaces xe-0/0/8 ether-options 802.3ad lacp force-up

#### Applicable only for QFX5K - Flow Control Configuration
set interfaces xe-0/0/8 ether-options flow-control
delete interfaces xe-0/0/8 ether-options flow-control

#### Applicable only for MX - Flow Control Configuration
set interfaces et-1/0/1 gigether-options flow-control
delete interfaces et-1/0/1 gigether-options flow-control

# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Testing
#### Unit tests
#### Dev tests
#### System tests

# 15. Documentation Impact
The feature and workflow will have to be documented

# 16. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-13247
Junos documentation: https://www.juniper.net/documentation/en_US/release-independent/nce/topics/example/evpn-vxlan-optional.html#jd0e139
                     https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/force-up-edit-interfaces-qfx-series.html)
                     https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/interfaces-edit-lacp.html
                     https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/flow-control-options-edit-interfaces.html
