# 1. Introduction
Manage configuration templates that he can apply on one or more of the managed devices.

- Visualize the list of templates in the repository, by name, device_family, date.

- edit/delete/upload of template

- Clone a template into a new version and edit it (e.g. add new User inputs, change the snippets)

- Create a New template and specify the User inputs for that template

APPLY THOSE TEMPLATES ON A SELECTED LIST OF DEVICES AND VERIFY IT HAS BEEN APPLIED CORRECTLY

- Select and apply on a list of devices
- Run the template, entering relevant inputs on the UI
Verify the configuration template has been applied properly on the device

### Supported Deployment model:
N/A

### Supported Device Models:
    QFX5120-32C​, QFX5120-48Y-8C, QFX5120-48T-6C, QFX10002-72Q/36Q/60C, QFX10008/QFX10016​

### Supported Device version:
    18.4R2-S3 and above.

# 2. Problem statement
CFM should support the Creation and deployment of Config template to the managed devices.

# 3. Proposed solution
The proposed solution will introduce
- a new api to validate and get the variables associated to the template. Create
a ConfigTemplate Object and call `push_config_template` job to push the configuration 
to the device

# 4. API schema changes
- New API called 'validate-template' to get variables associated to the template.
by providing name, device family, template content and type.

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
- Workflow for creation of template by name, device family, template content, type and variables.
- Grid to list all the templates created.
- Workflow to deploy config template to managed devices by providing global and device level parameters.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
- A new api to validate and get the variables associated to the template.
- New changes in fabric.xsd for ConfigTemplate object creation with template-name, template-content, device-family, template-type (SET or XML), device-vars and global-vars.
- New `push_config_template` which calls ansible role `push_config_template` introduced to push template content to 
managed devices according to provided global/device variables.

# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing

## 15.1 Sanity tests
1. Call 'validate-template' api by providing name, device family, template content and type.
2. Create ConfigTemplate object by providing name, device family, template content and type and parameters. 
3. Call `push_config_template` job with global parameters for both devices.
4. Job should be succeeded and config correct config should be pushed.

## 15.2 Feature tests
Below is the link to the feature test plan,
https://docs.google.com/spreadsheets/d/12wyEJqUX16oSHrBKCLIiv0F4Aa3KLLjg/edit?dls=true

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# Known issues and Caveat
There are no known issues for this functionality.

# 18. References
https://contrail-jws.atlassian.net/browse/CEM-15726


