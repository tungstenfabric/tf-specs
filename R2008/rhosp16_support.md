# 1. Introduction
RHOSP16 is LTS RedHat Openstack release based on Openstack Train.
Contrail Networking supports only RHOSP13 for now, so it 
is needed to provide support of RHOSP16 with Contrail Networking.

# 2. Problem statement

- RHOSP16 is deployed on RHEL8 based overcloud images that has:
  - new kernel
  - new container orchestration approach (docker is replaced with podman)
  - NetworkManager is default approach for network management and there is no ifup scripts 
- RHOSP16 has new heat templates layout and many changes in services 
and roles comparing to RHOSP13.

# 3. Proposed solution

- Use Contrail containers based on RHEL7 as they can be run on RHEL8
- For network management there is still ifup scripts are used, so
it is possible to use same approach with ifup as in RHOSP13.
- It is needed to compile and add appropriate vrouter module
 into the Contrail build that can be run on RHEL8
- Contrail TripleO Heat Templates are to be refactored to support new
layout, types, etc.
- Contrail TripleO Puppet module is to be revised to identify requrie changes if any

# 4. Implementation

- Compile vrouter for RHEL8 kernel
- Add new module into vrouter RPMs during build
- Support new heat templates laytout in Contrail TripleO Heat temapltes
- Make change in Contrail TripleO Puppet module to support changes in Train neutron config
(if any)
- Implement in Nodemanager ability to request containers info via podman
- Implement in contrail status tool ability to request containers info via podman
- Implement in agent ability to run conainers via podman instead of docker
(.//agent/oper/docker_instance_adapter.* )

# 5. Performance and scaling impact

N/A

# 6. Upgrade

Upgrade will be possible only via RedHat Fast Forward Upgrade with.
But at this step it is out ot the scope of this feature.
It will be handled by separate story after RedHat releases FFU procedure which 
will be a base for Contrail procedure.


# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

Testing is the same as for RHOSP13 (excluding upgrade as only new deploy now 
be supported).

# 10. Documentation Impact

Documentation about any new configuration knobs in
Contrail TripleO Heat Templates project.

# 11. References
