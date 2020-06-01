# 1. Introduction
This blueprint describes flow to virtual network mapping as part of slfow
integration.

# 2. Problem statement
Whe Logical Router (LR) is involved in the path of a flow from Bare Metal Server
(BMS) to Vrtual Machine (VM) hosted on Contrail Compute, in the flow, we get LR
interval Virtual Network (VN) as destination Virtual Network, in stead of tenant
Virtual Network, so user is not able to map from tenant VN which user has
created with the flow.

# 3. Proposed solution
Add an extra field for tenant Virtual Network both for source and destination in
Sandesh message.

# 4. API schema changes
In Sandesh, add an extra field to identify tenant virtual network.

# 5. Alternatives considered
As an alternative, the following approach can be used from insights flows side –
1. Customer queries the flow details between 2 customer vn.
2. Insights flows figures out the connecting LR vn for those 2 customer vn.
3. It sends 2 queries to get flow details – vn1 to LR vn and LR vn to vn2

Issue with the above approach is as below:
We need to make 2 queries, Query1 with Src VN: VN1 & Dest VN: LRVN and
Query2: Src VN: LRVN and DestVN: VN2,  and then merge the data and then
compute pkts and bytes which will be pretty slow if there are millions of records.
This will degrade the performance.
 

# 6. UI changes / User workflow impact
N/A

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
### Schema Changes
Add an extra field src_tenant_vn
```
@@ -298,6 +298,8 @@ struct FlowInfo {
    47: u32 rpf_nh;
    48: u32 src_ip_nh;
    49: u32 hbs_intf_dir;
+   50: optional list<string> tenant_source_vn_list;
+   51: optional list<string> tenant_dest_vn_list;
 }
```
### Agent Changes
Customer Virtual Network information needs to be propagated from agent to controller in LR vrf

### Controller Changes

# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A

# 14. Security Considerations
N/A

# 15. Testing
#### Unit tests
#### Dev tests
#### System tests

# 16. Documentation Impact

# 17. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-12861
