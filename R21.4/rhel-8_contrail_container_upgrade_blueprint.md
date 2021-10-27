# 1. Introduction

- The purpose of this document is to describe how contrail containers are getting build on RHEL-8 through Jenkins Job.

- Blueprint/Feature Lead: Sayyed Jahangir<sjahangir@juniper.net>

- Core team: 

  -Anil Chandra Raut<araut@juniper.net>

  -Fareed Shaik<fshaik@juniper.net>
 
- Jira Story: <https://contrail-jws.atlassian.net/browse/CEM-24546>

# 2. Problem statement

Currently contrail containers are based on RHEL7 and upgrading to RHEL8 is not supported. This document is to track the contrail container building on top of RHEL8 through Jenkins job.

# 3. Proposed solution

Master branch replication is required. R21.4 will be the new branch.

1. Creation a new pipeline job for the newly created branch in Jenkins by following the below links.
2. Modification of the contrail-build-scripts.
3. This proposed solution will support jenkins pipeline for RHEL-8, but will not support RHEL-7.

## 3.1 Limitations

N/A

## 3.2 Affected Modules

N/A

## 3.3 Alternatives considered

N/A

## 3.4 API schema changes

N/A

## 3.5 User workflow impact

To be updated

## 3.6 UI Changes

N/A

## 3.7 Operations and Notification impact

N/A

# 4. Implementation

- Need to have a Redhat satellite account is available for registration and subscription.

- Create Satellite content view and activation key for newly created branch https://engprod-svlsat2.contrail.juniper.net/

- Go to content -> content views -> select the source branch ->action ->copy content view (where it will ask for the new name,e.g;Master→R21.4)

- Publish a new version and Copy activation key from base branch to the targeted branch (paste in activate account key)

- Enable the required Rhel8 repos in redhat satellite
		rhel-8-for-x86_64-baseos-rpms
		rhel-8-for-x86_64-appstream-rpms
		rhel-8-for-x86_64-appstream-debug-rpms
		codeready-builder-for-rhel-8-x86_64-rpms
		openstack-16.2-for-rhel-8-x86_64-rpms
	
- Make sure the new branch is added in gitlab as R21.4 as input of “ Branch Name” and Master as input for “Create From”.

- Modify the “contrail-build-script/” for the newly created branch in GITLAB as per requirement.
 
  -- /nightly/containers/ubi.common.env.sample
	 Content needs to change for ubi8 as below
	 LINUX_DISTR="registry.access.redhat.com/ubi7/ubi"
	 LINUX_DISTR_VER="7.9-372"

  -- /nightly/jenkins-slave-image/rhel/Dockerfile
	 rhel8 version needs to update here
	 FROM registry.access.redhat.com/rhel7:7.8-265

  -- /nightly/jenkins-slave-image/rhel/rhel/redhat-mirror.repo
	 repos(rpms) for rhel8 needs to add here.
	 [master-rhel-7-server-rpms]
	 baseurl = http://pulp-server.englab.juniper.net/pulp/repos/master/rhel-7-server-rpms/
	 enabled = 1
	 gpgcheck = 0
	 name = rhel-7-server-rpms

  -- /nightly//build-test-containers-ubi.sh
	 sed command needs to update
	 sed -i 's#centos:7.4.1708#registry.access.redhat.com/ubi7/ubi:7.9-193#g' /home/contrail-builder/sandbox/third_party/contrail-test/dock         er/base/Dockerfile

  -- /nightly/commonLib.sh
	 rhel8 repos needs to add.

  -- A new pipeline job will be created for the newly created branch in Jenkins by following the below links
     https://cs-builder.englab.juniper.net/view/all/job/Nightly-Pipeline-Creation/

  -- Trigger the build (R21.4)by “Build with Parameters” after all above steps completed.
     Input for the CONFIG_BRANCH is “R21.4” which is newly created in GITLAB and the input for the “BUILD_BRANCH” is "Master".

  -- Make sure to create a new pipeline job for the newly created branch in Jenkins by following the below links
     https://cs-builder.englab.juniper.net/view/all/job/Nightly-Pipeline-Creation/

  -- Trigger the build by “Build with Parameters” after all above steps completed.

## 4.1 Assignee(s)

-  Sayyed Jahangir<sjahangir@juniper.net>
-  Anil Chandra Raut<araut@juniper.net>
-  Fareed Shaik<fshaik@juniper.net>

## 4.2 Work items

To be Updated

# 5. Performance and scaling impact

N/A

## 5.1 API and control plane

N/A

## 5.2 Forwarding performance

N/A

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

- Update UBI used for contrail container images https://contrail-jws.atlassian.net/browse/CEM-22821
- RHEL version should be above 8.2

# 9. Testing

To be Updated

# 10. Documentation Impact

The features overview, usage and its limitation should be documented as part of release documentation.

# 11. References

- https://contrail-jws.atlassian.net/browse/CEM-22821

