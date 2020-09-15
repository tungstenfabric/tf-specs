# 1. Introduction
This blueprint describes integrating Contrail and Appformix as a
single application for a unified telemetry user experience. With
this feature, Contrail can now provision network devices with telemetry
configurations and also have appformix related targets (eg. SNMP
targets, gRPC collectors and sflow collectors) under this telemetry
umbrella.

1. We support gRPC, SNMP, Netconf and JTI targets in addition to sflow, under
telemetry in R2011. The above sub-profiles will collectively termed as subtending profiles
for the remainder of this document.
2. JTI protocol metrics are collectively combined from both SNMP and Netconf.
Therefore JTI profile is not separately realized as a separate subtending profile
under the telemetry profile.

# 2. Problem statement
Once devices are discovered in Contrail Command, user should be able to
attach a telemetry profile to the devices. This telemetry profile
would be an encapsulation of other configs from the subtending profiles.

# 3. Proposed solution
The existing option, Infrastructure >  Fabrics > 'Telemetry Profiles' tab
can be used for the new subtending profiles as well.

The telemetry profile can be attached to a Physical Router (device).
There can be only one telemetry profile per device. The options to create
telemetry profiles, view existing ones are reused.

The telemetry profile will include a side panel to display the existing
subtending profiles. This will be in addition to the existing sflow profile view.

There can only be one instance of each of the different subtending profiles
(like sflow, gRPC, SNMP and/or Netconf) under a telemetry profile. For
instance a telemetry profile can still have both sflow profile and gRPC
profile but only one object of each.

The subtending profiles can be created at the time a telemetry
profile is created and referred to this telemetry profile. Otherwise
the user can choose to attach one or more existing subtending
profiles after the telemetry profile is created.

The gRPC profile will contain the following configuration parameters -

1. allow clients (cidr): default: 0.0.0.0/0 for default profiles and default value of
   appformix vip for user defined profiles
2. Physical(Environmental) Health (boolean(True/ False)): default: True
3. Interface Health (boolean(True/ False)): default: True
4. Control Plane Health (boolean(True/ False)): default: True

Rest of the other protocol based profiles such as the Netconf
and SNMP will have the following configuration parameters -

1. Physical(Environmental) Health (boolean(True/ False)): default: True
2. Interface Health (boolean(True/ False)): default: True
3. Control Plane Health (boolean(True/ False)): default: True

# 4. API schema changes

Objects to represent the Netconf, gRPC profile and the SNMP profile are
added to the fabric schema. The schema changes are captured below:

```
<xsd:complexType name="EnabledSensorParams">
    <xsd:all>
        <xsd:element name="physical-health" type="xsd:boolean" default="true"/>
        <xsd:element name="interface-health" type="xsd:boolean" default="true"/>
        <xsd:element name="control-plane-health" type="xsd:boolean" default="true"/>
        <xsd:element name="service-layer-health" type="xsd:boolean" default="true"/>
    </xsd:all>
</xsd:complexType>

<xsd:simpleType name="SecureModeType">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="cleartext"/>
         <xsd:enumeration value="ssl"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="gRPCParameters">
    <xsd:all>
       <xsd:element name="allow-clients" type="SubnetListType" required="true"
            description= "whitelist subnet of allowed clients on which various KPIs can be monitored." />
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. The user can include one or all of these for telemetry monitoring by sele
       <xsd:element name="secure-mode" type="SecureModeType" required="optional"
            description= "secure (SSL) or cleartext mode of gRPC configuration." />
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="snmpParameters">
    <xsd:all>
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. The user can include one or all of these for telemetry monitoring by sele
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="netconfParameters">
    <xsd:all>
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. The user can include one or all of these for telemetry monitoring by sele
    </xsd:all>
</xsd:complexType>

<xsd:element name="telemetry-profile-grpc-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-grpc-profile', 'telemetry-profile', 'grpc-profile', ['ref'], 'optional', 'CRUD',
             'gRPC profile that this telemetry profile uses. Only one gRPC profile can be associated to one telemetry profile.') -->
<xsd:element name="telemetry-profile-netconf-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-netconf-profile', 'telemetry-profile', 'netconf-profile', ['ref'], 'optional', 'CRUD',
             'netconf profile that this telemetry profile uses. Only one netconf profile can be associated to one telemetry profile.') -->
<xsd:element name="telemetry-profile-snmp-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-snmp-profile', 'telemetry-profile', 'snmp-profile', ['ref'], 'optional', 'CRUD',
             'snmp profile that this telemetry profile uses. Only one snmp profile can be associated to one telemetry profile.') -->

<xsd:element name="grpc-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to grpc parameters"/>

<xsd:element name="project-grpc-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-grpc-profile', 'project', 'grpc-profile', ['has'], 'optional', 'CRUD',
             'list of grpc profiles supported under the project.') -->

<xsd:element name="grpc-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('grpc-profile-is-default', 'grpc-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default grpc profile or not. Default profiles are non-editable.') -->

<xsd:element name="grpc-parameters" type="gRPCParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('grpc-parameters', 'grpc-profile', 'optional', 'CRUD',
                  'Parameters for each grpc profile, such as allow client, top level sensor options etc.') -->

<xsd:element name="snmp-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to snmp parameters"/>

<xsd:element name="project-snmp-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-snmp-profile', 'project', 'snmp-profile', ['has'], 'optional', 'CRUD',
             'list of snmp profiles supported under the project.') -->

<xsd:element name="snmp-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('snmp-profile-is-default', 'snmp-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default snmp profile or not. Default profiles are non-editable.') -->

<xsd:element name="snmp-parameters" type="snmpParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('snmp-parameters', 'snmp-profile', 'optional', 'CRUD',
                  'Parameters for each snmp profile like the top level sensor options etc.') -->

<xsd:element name="netconf-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to netconf parameters"/>

<xsd:element name="project-netconf-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-netconf-profile', 'project', 'netconf-profile', ['has'], 'optional', 'CRUD',
             'list of netconf profiles supported under the project.') -->

<xsd:element name="netconf-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('netconf-profile-is-default', 'netconf-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default netconf profile or not. Default profiles are non-editable.') -->

<xsd:element name="netconf-parameters" type="netconfParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('netconf-parameters', 'netconf-profile', 'optional', 'CRUD',
                  'Parameters for each netconf profile like the top level sensor options etc.') -->

```

# 5. Notification impact
N/A

# 6. Backend Implementation
The main implementation changes include:
1) Create new objects to represent the subtending profiles in the schema.

2) Device manager changes to cache these new objects

3) Device manager changes to generate abstract config changes upon creation,
updation and deletion of the telemetry and gRPC objects

4) Jinja templates to generate the telemetry configuration

### The gRPC configurations for QFX10K and QFX5K are captured below:

set system services extension-service request-response grpc clear-text port 50051
set system services extension-service request-response grpc skip-authentication
set system services extension-service notification allow-clients address 10.0.0.0/24

# 7. Performance and scaling impact
N/A

# 8. Upgrade
N/A

# 9. Deprecations
N/A

# 10. Dependencies
N/A

# 11. Testing
#### Unit tests
#### Dev tests
#### System tests

# 12. Documentation Impact
The feature and workflow will have to be documented

# 13. References
JIRA stories : https://contrail-jws.atlassian.net/browse/CEM-17285
               https://ssd-git.juniper.net/appformix/AppFormix/-/wikis/appformix-grpc-network-device-telemetry#login-pane

# 14. Alternatives considered
For user input of allow_clients for gRPC profile, the following ideas were considered, but
were eventually rejected due to the reasons mentioned aside
1. allow clients cidr could be input during provisioning just like xflow node collector ip at present.
However, allow clients need not represent only appformix cidr, could be anything like a homegrown solution
or a compute cidr wherever appformix is deployed.
2. For scaling, there could be potentially many network agents which could be deployed anywhere. So the allow
clients information need to be captured at a point just before telemetry profile is attached to a device.
3. We could use the CC to derive the information from the GO-API during provisioning internally. However
the attachment of a telemetry profile to a device comes much later and hence the user may wish to configure
the parameters differently.

