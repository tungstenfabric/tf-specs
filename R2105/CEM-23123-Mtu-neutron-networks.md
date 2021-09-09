# 1. Introduction

- The purpose of this document is to describe how MTU property is  
  supported and implemented within VirtualNetwork resources.  
  This allows any user to be configured from the  
  Orchestration service (OpenStack) and via REST API interfaces.

- Blueprint/Feature Lead: Nagendra Maynattamai ( <npchandran@juniper.net> )

- Core team:  

  - Shuvankar Chakraborty ( <shuvankarc@juniper.net> )
  - Prashanth K R ( <pkr@juniper.net> )
  - Sriram Natarajan ( <snatarajan@juniper.net> )

- Jira Story: <https://contrail-jws.atlassian.net/browse/CEM-23123>

# 2. Problem statement

Openstack neutron network resource allows to configure MTU for a given
network. Contrail should consume the property and store it in the
VirtualNetwork resource and persist its configured value.  
At this point of time, the mtu value configured for a virtual network
resource will be just saved in Contrail config. No further processing
will be done by other components i.e Control or Agent.

# 3. Proposed solution

1. Enable neutron plugins to allow configure and update MTU property  
   in neutron networks

2. Update contrail neutron-plugin to consume mtu value from neutron  
   server and reconfigure it in contrail virtual network resource

3. Update vnc-openstack where required to include mtu property

4. Update contrail datamodel schema to include MTU as a virtual network  
   property

5. Update Contrail UI to allow configuring and updating MTU property for a  
   given virtual network

## 3.1 Limitations

1. This implementation do not support consuming MTU value of virtual networks other than configure, update and persists the configured value in Contrail Config. No support will be provided to use this value in DHCP configuration of new workloads.

## 3.2 Affected Modules

1. Contrail Neutron Plugin

2. Contrail Config (vnc-openstack, API-Server)

3. Deployment (support configuring mtu plugin at neutron.conf, ContrailPlugin.ini)

4. Contrail UI

5. Heat templates

## 3.3 Alternatives considered

N/A

## 3.4 API schema changes

1. Include mtu attribute in virtualnetwork properties

2. Setup maximum and minimum allowed values

## 3.5 User workflow impact

To be updated

## 3.6 UI Changes

1. User should be able to configure virtual network resource without any  
   mtu value (default behavior)

2. Provide an option to configure MTU value in virtual network resource

3. Provide an option to update MTU value in virtual network resource

## 3.7 Operations and Notification impact

N/A

# 4. Implementation

To be Updated

## 4.1 Assignee(s)

- Nagendra Maynattamai ( <npchandran@juniper.net> )

- Shuvankar Chakraborty ( <shuvankarc@juniper.net> )

- Sriram Natarajan ( <snatarajan@juniper.net> )

- Prashanth K R ( <pkr@juniper.net> )

## 4.2 Work items

To be Updated

# 5. Performance and scaling impact

## 5.1 API and control plane

*Scaling and performance for API and control plane*

To be updated

## 5.2 Forwarding performance

To be added

# 6. Upgrade

1. Existing Virtual networks from older release should include default MTU value i.e None

2. MTU value can be configured on a virtual network after upgrade

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

## 9.1 Unit tests

1. Unit tests coverage for schema changes
2. Unit tests in API-Server module to validate configure/update MTU property
3. Unit tests in VNC-openstack module to validate configure/update MTU property

## 9.2 Dev tests

1. Verify a virtual network can be created without mtu property
2. Verify a virtual network can be created with allowed mtu value range
3. Verify mtu value configured in virtual network can be updated
4. Verify mtu value can be reverted back to default (i.e None)
5. Verify no workloads are affected by mtu value configured in virtual networks
6. From Openstack, Verify a virtual network can be created without mtu property
7. From Openstack, Verify a virtual network can be created with allowed mtu value range
8. From Openstack, Verify mtu value configured in virtual network can be updated
9. From Openstack, Verify mtu value can be reverted back to default (i.e None)

## 9.3 Upgrade tests

1. Verify virtual networks in all supported upgrade path are unaffected and  
   configured with default mtu (i.e None)

2. Verify virtual networks from old release can be configured with mtu value after upgrade

## 9.3 UI tests

To be added

# 10. Documentation Impact

The features overview, usage and its limitation should be documented as part of release documentation.  

# 11. References

To be updated
