# TFF-18 Logical Router Type

## 1. Introduction
The TF config api supports logical router creation with custom type, either vxlan-routing or snat-routing. This blueprint adds that feature to the TF webui.

## 2. Problem statement
Since the TF config api already supported creating logical routers with their own type and is independent from global project configuration, we should add that function to the TF WebUI so that users can specify which mode they want them to operate on.


## 3. Proposed solution

### Backgrounds
A logical router can be created by selecting the Config > Network > Routers section, then clicking the "+" (Create Routers) button.

### Solution
- Adding a selection box providing two options for user to choose from, which are 'snat-routing' and 'vxlan-routing' separately.
- The default choice is snat-routing.
- The previously frozen "SNAT" checkbox is removed to avoid confusions.

## 4. API schema changes
None


## 5. Alternatives considered
N/A


## 6. UI changes / User workflow impact

### UI changes
The logical router creation form has changed: The frozen SNAT checkbox is replaced by the logical router's type selection box.


## 7. Notification impact
N/A


## 8. Provisioning changes
N/A


## 9. Implementation
- Added a new field to the logical router model in the UI: "`logical_router_type`"
- This field is passed along with the CREATE request to the config api. The config api then creates a new router with the chosen type.

## 11. Performance and scaling impact
N/A


## 12. Upgrade
N/A


## 13. Deprecations
N/A


## 14. Dependencies
N/A


## 15. Testing
N/A


## 16. Documentation Impact
It will be documented as part of release documentation by the doc team.


## 17. Known issues and Caveat
There are no known issues for this functionality.


## 18. References
[TFF-18](https://jira.tungsten.io/browse/TFF-18?src=confmacro)
