# 1. Introduction
This blueprint describes integrating Contrail and Appformix as a
single application for a unified telemetry user experience. With
this feature, Contrail can now provision network devices with telemetry
configurations and also have appformix related targets (eg. SNMP
targets, gRPC collectors and sflow collectors) under this telemetry
umbrella.

1. Both in-band and out-of-band(OOB) appformix flow provisioning is supported in R1911.
2. Appformix flow nodes can be provisioned only once and before fabric on-boarding.
3. No new appformix flow nodes can be added to the cluster after initial provisioning is done.
4. Appformix flow nodes can be connected to only one TOR.
5. Also, we support gRPC, SNMP, netconf and JTI targets in addition to sflow, under
telemetry in R2011. The above sub-profiles will collectively termed as subtending profiles
for the remainder of this document.
6. Dashboard for alarms and port statuses including various KPIs is also supported
in R2011.

# 2. Problem statement
Once devices are discovered in Contrail Command, user should be able to
attach a telemetry profile to the devices. This telemetry profile
would be an encapsulation of other configs from the subtending profiles.

# 3. Proposed solution
A new Infrastructure >  Fabrics > 'Telemetry Profiles' tab will be added.

The telemetry profile can be attached to a Physical Router (device).
There can be only one telemetry profile per device. The options to create
telemetry profiles, view existing ones will be provided.

The telemetry profile will include a side panel to display the existing
subtending profiles.

There can only be one instance of each of the different subtending profiles
(like sflow, gRPC, SNMP, netconf and/ or JTI) under a telemetry profile. For
instance a telemetry profile can still have both sflow profile and gRPC
profile but only one object of each.

The subtending profiles can be created at the time a telemetry
profile is created and referred to this telemetry profile. Otherwise
the user can choose to attach one or more existing subtending
profiles after the telemetry profile is created.

The sflow profile will contain the following configuration parameters -
sample rate, polling interval, agent id, adaptive sample rate, enabled
interface type and enabled interface params.

The following are the accepted values for some of the sflow parameters:
1. sample rate (number (one packet out of number)): Accepted Range: 1-16777215, default: 2000
2. polling interval (seconds): Accepted Range: 0-3600(seconds), default: 20
3. adaptive sample rate (number): Accepted Range: 300-900, default: 300

The gRPC profile will contain the following configuration parameters -
1. gRPC enable (boolean(True/ False)): default: False
2. allow clients (cidr): default: appformix flow_collector_vip_ip
3. Physical(Environmental) Health (boolean(True/ False)): default: True
4. Interface Health (boolean(True/ False)): default: True
5. Control Plane Health (boolean(True/ False)): default: True

Rest of the other protocol based profiles such as the netconf,
JTI and SNMP will have the following configuration parameters -

1. Physical(Environmental) Health (boolean(True/ False)): default: True
2. Interface Health (boolean(True/ False)): default: True
3. Control Plane Health (boolean(True/ False)): default: True

The Alarm count display depicts the number of different categories of alarms
(Critical/ Major/ Minor) across fabrics (inter-fabric). Similarly the port
statuses also depicts the different port status (up/ down) across fabric.
These data are obtained from appformix sensors enabled on the devices on
fabrics. Clicking a particular fabric would show the same information but
only in context of the fabric which was selected (intra-fabric).

The device operational status will have the following values:
1. Up: As long as the status falls within the threshold of the sensor(s) set
2. Down: Any status that exceeds the threshold on the sensor (includes both
at risk and down criteria)

The administrative status on a device will carry forward the statuses as used
by the contrail fabric currently.

# 4. API schema changes

Each appformix flow node is represented as a flow-node object in VNC database.
Objects to represent collector parameters such as virtual IP address
are added to the vnc schema. Underlay telemetry infra subnet is modeled for supporting
in-band collector provisioning. The schema changes are captured below:

```
<xsd:element name="instance-ip-flow-node"/>
<!--#IFMAP-SEMANTICS-IDL
     Link('instance-ip-flow-node',
          'instance-ip', 'flow-node', ['ref'], 'optional', 'CRUD',
          'Reference to the flow node that this instance ip is assigned to.') -->

<xsd:element name="flow-node" type="ifmap:IdentityType"/>
<xsd:element name="global-system-config-flow-node"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('global-system-config-flow-node',
             'global-system-config', 'flow-node', ['has'], 'admin-only', 'CRUD',
             'Appformix flows node is object representing a logical node in system which serves xflow collectors.') -->

<xsd:element name="flow-node-ip-address" type="IpAddressType"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('flow-node-ip-address', 'flow-node', 'admin-only', 'CRUD',
              'Ip address of the appformix flow node, set while provisioning.') -->

<xsd:element name="flow-node-virtual-network"/>
<!--#IFMAP-SEMANTICS-IDL
         Link('flow-node-virtual-network',
             'flow-node', 'virtual-network', ['ref'], 'optional', 'CRUD',
             'Similar to using virtual-machine to model the bare metal server, we are using virtual-network to model telemetry underlay infra network. This would allow us to re-use the same IPAM data model and code base to manage the IP auto-assignments for the underlay telemetry networks.') -->

<xsd:element name="flow-node-load-balancer-ip" type="IpAddressType"/>
<!--#IFMAP-SEMANTICS-IDL
         Property('flow-node-load-balancer-ip', 'flow-node', 'required', 'CRUD',
              'IP address of the load balancer node for xflow collectors, set while provisioning.') -->

<xsd:element name="flow-node-inband-interface" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
         Property('flow-node-inband-interface', 'flow-node', 'optional', 'CRUD',
              'In-Band interface name used for this flow node.') -->
```

Objects to represent the telemetry profile, gRPC profile and the sflow profile are
added to the fabric schema. The schema changes are captured below:

```
<xsd:simpleType name="SampleRate">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="1"/>
        <xsd:maxInclusive value="16777215"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name="PollingInterval">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="0"/>
        <xsd:maxInclusive value="3600"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:simpleType name="AdativeSampleRate">
    <xsd:restriction base="xsd:integer">
        <xsd:minInclusive value="300"/>
        <xsd:maxInclusive value="900"/>
    </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="StatsCollectionFrequency">
    <xsd:all>
        <xsd:element name="sample-rate" type="SampleRate" required="optional" default="2000"/>
        <xsd:element name="polling-interval" type="PollingInterval" required="optional" default="20"/>
    </xsd:all>
</xsd:complexType>

<xsd:simpleType name="EnabledInterfaceType">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="all"/>
         <xsd:enumeration value="fabric"/>
         <xsd:enumeration value="service"/>
         <xsd:enumeration value="access"/>
         <xsd:enumeration value="custom"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="EnabledInterfaceParams">
    <xsd:all>
        <xsd:element name="name" type="xsd:string"/>
        <xsd:element name="stats-collection-frequency" type="StatsCollectionFrequency"/>
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="SflowParameters">
    <xsd:all>
       <xsd:element name="stats-collection-frequency" type="StatsCollectionFrequency"
            description= "Represents polling interval and sample rate either at the global level or at per interface level. Polling interval is specified in seconds that the device waits between port statistics update messages. Sample rate is specified as a number (one packet out of number)." />
       <xsd:element name="agent-id" type="smi:IpAddress"
            description= "IP address to be assigned as the agent ID for the sFlow agent." />
       <xsd:element name="adaptive-sample-rate" type="AdativeSampleRate" required="optional" default="300"
            description= "Represents  maximum number of samples that should be generated per line card." />
       <xsd:element name="enabled-interface-type" type="EnabledInterfaceType"
            description= "User can enable sflow either on all interfaces or all fabric ports or all access ports or custom list of interfaces." />
       <xsd:element name="enabled-interface-params" type="EnabledInterfaceParams" maxOccurs="unbounded"
            description= "If interface type is set to custom, this represents the list of physical interfaces to be enabled for sflow. User can specify polling interval and sample rate for each interface in this custom list." />
    </xsd:all>
</xsd:complexType>

<xsd:element name="telemetry-profile" type="ifmap:IdentityType"
    description="Encapsulates data related to telemetry from network devices like sflow, JTI, gRPC, SNMP etc"/>

<xsd:element name="project-telemetry-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-telemetry-profile', 'project', 'telemetry-profile', ['has'], 'optional', 'CRUD',
             'list of telemetry profiles supported under the project.') -->
<xsd:element name="telemetry-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('telemetry-profile-is-default', 'telemetry-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default telemetry profile or not. Default profiles are non-editable.') -->

<xsd:element name="telemetry-profile-sflow-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-sflow-profile', 'telemetry-profile', 'sflow-profile', ['ref'], 'optional', 'CRUD',
             'Sflow profile that this telemetry profile uses. Only one sflow profile can be associated to one telemetry profile.') -->
<xsd:element name="telemetry-profile-grpc-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-grpc-profile', 'telemetry-profile', 'grpc-profile', ['ref'], 'optional', 'CRUD',
             'gRPC profile that this telemetry profile uses. Only one gRPC profile can be associated to one telemetry profile.') -->
<xsd:element name="telemetry-profile-netconf-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-netconf-profile', 'telemetry-profile', 'netconf-profile', ['ref'], 'optional', 'CRUD',
             'netconf profile that this telemetry profile uses. Only one netconf profile can be associated to one telemetry profile.') -->
<xsd:element name="telemetry-profile-jti-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-jti-profile', 'telemetry-profile', 'jti-profile', ['ref'], 'optional', 'CRUD',
             'jti profile that this telemetry profile uses. Only one jti profile can be associated to one telemetry profile.') -->
<xsd:element name="telemetry-profile-snmp-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('telemetry-profile-snmp-profile', 'telemetry-profile', 'snmp-profile', ['ref'], 'optional', 'CRUD',
             'snmp profile that this telemetry profile uses. Only one snmp profile can be associated to one telemetry profile.') -->
<xsd:element name="physical-router-telemetry-profile"/>

<!--#IFMAP-SEMANTICS-IDL
          Link('physical-router-telemetry-profile', 'physical-router', 'telemetry-profile', ['ref'], 'optional', 'CRUD',
             'Telemetry profile assigned to the physical router by user. Each physical router is associated with only one telemetry profile.') -->

<xsd:element name="sflow-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to sflow parameters"/>

<xsd:element name="project-sflow-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-sflow-profile', 'project', 'sflow-profile', ['has'], 'optional', 'CRUD',
             'list of sflow profiles supported under the project.') -->
<xsd:element name="sflow-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('sflow-profile-is-default', 'sflow-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default sflow profile or not. Default profiles are non-editable.') -->

<xsd:element name="sflow-parameters" type="SflowParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('sflow-parameters', 'sflow-profile', 'optional', 'CRUD',
                  'Parameters for each sflow profile, such as polling interval, sample rate, list of sflow enabled interfaces, sflow agent ID etc.') -->

<xsd:simpleType name="EnabledSensorParams">
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="physical-health"/>
         <xsd:enumeration value="interface-health"/>
         <xsd:enumeration value="control-plane-health"/>
         <xsd:enumeration value="service-layer-health"/>
     </xsd:restriction>
</xsd:simpleType>

<xsd:complexType name="gRPCParameters">
    <xsd:all>
       <xsd:element name="grpc-enable" type="xsd:boolean" default="false"
            description= "if this option is selected, then this enables gRPC based metric collection on select sensors" />
       <xsd:element name="allow-clients" type="SubnetListType" required="optional" default="0.0.0.0/0"
            description= "whitelist subnet of allowed clients on which various KPIs can be monitored. This option is available only if grpc-enable is checked (True)" />
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams" maxOccurs="unbounded"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. These options are all enabled by default when the grpc-enable is set to True. The user can exclude from telemetry monitoring by deselecting specific options." />
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="jtiParameters">
    <xsd:all>
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams" maxOccurs="unbounded"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. The user can include one or all of these for telemetry monitoring by selecting specific options." />
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="snmpParameters">
    <xsd:all>
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams" maxOccurs="unbounded"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. The user can include one or all of these for telemetry monitoring by selecting specific options." />
    </xsd:all>
</xsd:complexType>

<xsd:complexType name="netconfParameters">
    <xsd:all>
       <xsd:element name="enabled-sensor-params" type="EnabledSensorParams" maxOccurs="unbounded"
            description= "List of different top level sensor params that the user wishes to monitor using telemetry. The user can include one or all of these for telemetry monitoring by selecting specific options." />
    </xsd:all>
</xsd:complexType>

<xsd:element name="grpc-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to grpc parameters"/>

<xsd:element name="project-grpc-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-grpc-profile', 'project', 'grpc-profile', ['has'], 'optional', 'CRUD',
             'list of grpc profiles supported under the project.') -->

<xsd:element name="grpc-parameters" type="gRPCParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('grpc-parameters', 'grpc-profile', 'optional', 'CRUD',
                  'Parameters for each grpc profile, such as grpc enable, allow client, top level sensor options etc.') -->
<xsd:element name="jti-profile" type="ifmap:IdentityType"
    description="This resource contains information specific to jti parameters"/>

<xsd:element name="project-jti-profile"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('project-jti-profile', 'project', 'jti-profile', ['has'], 'optional', 'CRUD',
             'list of netconf profiles supported under the project.') -->

<xsd:element name="jti-profile-is-default" type="xsd:boolean" default="false"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('jti-profile-is-default', 'jti-profile', 'optional', 'CRUD',
              'This attribute indicates whether it is a default jti profile or not. Default profiles are non-editable.') -->

<xsd:element name="jti-parameters" type="jtiParameters"/>
<!--#IFMAP-SEMANTICS-IDL
              Property('jti-parameters', 'jti-profile', 'optional', 'CRUD',
                  'Parameters for each jti profile like the top level sensor options etc.') -->

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

# 5. UI changes / User workflow impact

The UI workflow differs in few respects for in-band and out-of-band
appformix flow node provisioning.

We can broadly classify these into 3 steps:
1. Appformix flow provisioning during cluster bring-up
2. Configuring Appformix collectors after fabric on-boarding
3. Creation of telemetry/subtending profiles and attachment of
telemetry profiles to devices in the fabric

### 1. Appformix flow provisioning during cluster bring-up

This varies a bit for in-band and out-of-band collector provisioning as
detailed below.

#### - In-Band Provisioning Workflow
The following inputs need to be provided for in-band collector provisioning
during cluster bring up:
a. CIDR: This is the underlay telemetry infra subnet. In-band interface on the appformix
flow node will be assigned an ip address from this subnet.
b. Vlan-id: To be used for the telemetry infra network.
c. Management Virtual IP Address: This is for contrail command to connect to appformix flow
node.
d. Show Advanced Options: User needs to choose specific options from this tab for in-band.
   - In-Band Virtual IP Address: This is usually defaulted to the third IP address from the
     CIDR (telemetry subnet). However, the user can choose to edit this as necessary. This will
     be used as collector destination ip for sflow target on TOR switches and for vRouter contrail flows.
e. Available Servers: Choose the server where appformix flow node will be installed.
   - In-band Interface: interface name on the server chosen above which is connected to the TOR.

#### - Out-Of-Band Provisioning Workflow
The following inputs need to be provided for out-of-band collector provisioning
during cluster bring up:
a. Virtual IP Address: This translates to IP address of the load balancer node for
xflow collectors
b. Available Servers: Choose the server where appformix flow node will be installed.


### 2. Configuring Appformix collectors after fabric on-boarding

This step is available for in-band collector provisioning only.
#### - Device Port Selection
The following information is pre-populated from the setup wizard (during
cluster bring up):
- In-band collector configuration
  a. Telemetry CIDR subnet
  b. Virtual IP Address
- In-band collector details
  a. Hostname (Server name)
  b. Management IP
  c. Interface Name

The user needs to select which device in the fabric to connect to the configured
server port.

### 3. Configuring static routes on Contrail vRouter agent to flow-collectors for in-band
As for in-band flow-collector provisioning, flow-collectors run on different subnet
than vRouter, so we need to add a static route in vRouter agent to reach to flow-collectors.


### 4. Creation of telemetry/subtending profiles and attachment of
telemetry profiles to devices in the fabric

This step is common for both modes of collector provisioning - in-band and OOB.

#### - Select Telemetry Profiles from Fabrics (under Infrastructure)
User selects the 'Telemetry Profiles' option under the fabric option

#### - Existing telemetry profiles list
The existing telemetry profile objects are also displayed above. User can create
new telemetry profiles from this page

#### - Existing sflow profiles list
Upon clicking to edit a telemetry profile or creating a new one, the user is
taken into this page. The sflow Properties option here displays all the existing
sflow profile objects. The user can just select a single existing
sflow profile or just create a new one via 'Create New' option on the side.

#### - Sflow control creation page
User can create a new sflow profile by entering the profile name,
sampling rate, polling interval and adaptive sample rate. The user
must mandatorily select one of the following interface types
1. All Revenue Interfaces
2. All Fabric Interfaces
3. All Interfaces
4. Custom (user can choose multiple devices from multiple fabrics and multiple
interfaces from each of the devices)

#### - Existing gRPC profiles list
Upon clicking to edit a telemetry profile or creating a new one, the user is
taken into this page. The gRPC Profile option here displays all the existing
gRPC profile objects. The user can just select a single existing
gRPC profile or just create a new one via 'Create New' option on the side.

#### - gRPC profile creation page
User can create a new gRPC profile by entering the profile name,
enable/disable gRPC, and the following if gRPC telemetry collection is enabled:
1. allow_clients cidr
2. check or uncheck top level sensors (Physical/ Interface/ ControlPlane) for collecting metrics.

The user must select if the gRPC based telemetry data collection should be enabled or
not. Also ideally if the user selects gRPC for telemetry data collection, then the
whitelist of allow clients cidr must be a meaningful subnet. If this information is
not provided, the subnet is default to virtual ip address from appformix. The user
has the liberty to override the allow clients cidr at any point of time except for
the default gRPC profile which is not editable. The user can also choose to clone
the default gRPC profile and tweak the cloned one to suit the intent.

The default gRPC profile will have the following parameters pre-configured:
1. enable gRPC: On
2. allow_clients: appformix flow_collector_vip_ip
3. all the top level health sensors enabled

#### - Existing JTI/ netconf/ SNMP profiles list
Upon clicking to edit a telemetry profile or creating a new one, the user is
taken into this page. The JTI/ netconf/ SNMP Profile options here display all
the existing subtending profile objects under the corresponding field name.
The user can just select an existing subtending profile or just create a new
one via 'Create New' option on the side.

#### - JTI/ netconf/ SNMP profile creation page
User can create a new JTI/ netconf/ SNMP profile by entering the profile name,
and check or uncheck top level sensors (Physical/ Interface/ ControlPlane) for collecting metrics.

The default profiles for each of the above sub profiles will
have all of the top level health sensors enabled automatically.

#### - Telemetry profile attachment to Physical Router
The telemetry profile can be assigned to the Physical Router in this page.
Navigate to select Fabric and click on the fabric. This will list the devices
in the fabric. User can choose to group devices to be assigned same telemetry
profile or assign different telemetry profiles to different devices.

Upon selecting the devices to be assigned a telemetry porfile, an option 'Assign
Telemetry Profile' will be enabled. Clicking this button will then display a
pop-up which will list all existing telemetry profiles. Also hovering over a
particular telemetry profile will show a side panel displaying the sflow and
gRPC details if attached to this telemetry profile.

# 7. Notification impact
N/A

# 8. Provisioning changes
During flow collector provisioning stage, the flow collector's details will be
registered to Contrail API Server. A POST request will be sent from flow collector
deployer to API Server with collector management and load-balancer IP address.
POST data format to API Server:
```
'flow-node': {
    'fq_name': ['default-global-system-config', '<flow_collector_fqdn>'],
    'parent_type': 'global-system-config',
    'flow_node_ip_address': '<flow_collector_ip>',
    'flow_node_load_balancer_ip': '<flow_collector_vip_ip>',
    'display_name': '<flow_collector_fqdn>',
    'name': '<flow_collector_fqdn>'
}
```
Section [API schema changes](https://github.com/Juniper/contrail-specs/blob/master/R1910/telemetry_feature_blueprint.md#4-api-schema-changes) describes schema changes for this

If `contrail_configuration.AAA_MODE` is set as `cloud-admin` or `rbac`, then
a request is being sent to keystone with `KEYSTONE_AUTH_ADMIN_USER`,
`KEYSTONE_AUTH_ADMIN_PASSWORD` and `KEYSTONE_AUTH_ADMIN_TENANT` to receive
keystone token. This token is then passed in `X-Auth-Token` header to
communicate with Contrail API SErver from flow collector deployer.

# 9. Backend Implementation
The main implementation changes include:
1) Create new objects to represent the telemetry profile, the sflow
profile and the gRPC profile in the schema. The telemetry profile will
refer to the Physical Router (device) object.

2) Device manager changes to cache these new objects

3) Device manager changes to generate abstract config changes upon creation,
updation and deletion of the telemetry(sflow, gRPC) objects

4) Jinja templates to generate the telemetry configuration

### The sflow configurations for QFX10K and QFX5K are captured below:

#### Applicable for QFX - To attach the sflow profile

set protocols sflow interfaces xe-0/0/0
set protocols sflow interfaces xe-0/0/0 sample-rate ingress 1200
set protocols sflow interfaces xe-0/0/0 sample-rate egress 1200
set protocols sflow interfaces xe-0/0/0 polling-interval 900

###### Configuring Profile:
set protocols sflow adaptive-sample-rate 500 fallback
set protocols sflow collector 10.1.1.0 udp-port 6343
set protocols sflow agent-id 5.5.5.5
set protocols sflow sample-rate ingress 300
set protocols sflow sample-rate egress 500
set protocols sflow polling-interval 1000

### The gRPC configurations for QFX10K and QFX5K are captured below:

set system services extension-service request-response grpc clear-text port 50051
set system services extension-service request-response grpc skip-authentication
set system services extension-service notification allow-clients address 10.0.0.0/24

# 10. UI/Appformix Implementation

Starting R2011, contrail command (CC UI) will consume the appformix api endpoints
for showing the various health KPIs and the port statuses, alarms dashboard.

UI will post requests to appformix API whenever a telemetry profile is attached
to a device after onboarding. UI will as well post requests to appformix in case
of an update either in the profile(s) properties itself or if the attachment to the
device changes. This is done to relay any sensor parameters changes and related
device information to and fro Appformix for real time data display.

# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing
#### Unit tests
#### Dev tests
#### System tests

# 16. Documentation Impact
The feature and workflow will have to be documented

# 17. References
JIRA stories : https://contrail-jws.atlassian.net/browse/CEM-4836
               https://contrail-jws.atlassian.net/browse/CEM-5819
In-band collector provisioning design document: https://docs.google.com/document/d/1JjJSrqSC08kNrzsbtaq_DMcyJelYmZ2eJk4mx9jAuvI/edit
Junos documentation: https://www.juniper.net/documentation/en_US/junos/topics/reference/configuration-statement/sflow-edit-protocols-qfx-series.html

# 18. Alternatives considered
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

