# Problem Statement

Combined FFU+ZIU upgrade from RHOSP13/CN19XX to RHOSP16.1/CN2011 LTS
Procedure should be based on RedHat FFU as much as possible.



# Proposed Solution and Implementation

Deploy virtual KVM lab with tf-devstack. Verify RedHat FFU procedure and reproduce all the steps in virtual KVM lab.
Add additional steps and workarounds for contrail (if needed).
Provide step-by-step manual


Standard RedHat FFU: https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.1/html/framework_for_upgrades_13_to_16.1/






# Acceptance Criteria
Successful upgrade in virtual KVM lab deployed with tf-devstack.
contrail-status is OK
Step-by-step upgrade manual provided


