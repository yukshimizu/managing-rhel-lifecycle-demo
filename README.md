# managing-rhel-lifecycle-demo
This repo includes ansible playbooks for a demo project of automating content management in Red Hat Satellite with Red Hat Ansible Automation Platform on AWS EC2.

## Automating Content Management in Red Hat Satellite with Red Hat Ansible Automation Platform
The goal behind the code is to demonstrate a simple example for automating the periodic publishing of Content Views in Red Hat Satellite and testing those contents before applying to production.

## Assumed demo environment
The assumed environment can be set up on AWS EC2 easily by using playbooks and roles in the [managing-rhel-lifecycle-seup](https://github.com/yukshimizu/managing-rhel-lifecycle-setup) paired repo. 

## Included contents
### Playbooks
|Name     |Description|
|:--------|:----------|
|`publish_cv.yml`|Publish a new version of the configured content view.|
|`promote_cv.yml`|Promote the content view to a life cycle environment.|
|`snapshot_vm.yml`|Snapshot the disk of runnning managed VMs.|
|`update_vm.yml`|Update all packages following the configured content view.|
|`restart_vm.yml`|Restart the running managed VMs.|
|`test_vm.yml`|Test the infrastructure of the restarted VMs.|
|`test_app.yml`|Test the application of the restarted VMs.|

### Group variables
These variables have already been set as follows, corresponding to the setup by [managing-rhel-lifecycle-setup](https://github.com/yukshimizu/managing-rhel-lifecycle-setup). You can adjust them based on your environment.
```
purpose: demo

satellite_organization: My_Organization
satellite_location: Tokyo
satellite_admin_username: admin
satellite_activation_key_name: "Demo_Key"
satellite_content_view: "RHEL9_SOE"
satellite_content_view_filter_types:
  - bugfix
  - security
  - enhancement

managed_vms_name_prefix: "managed"
aws_region: ap-northeast-1 # adjust with your preference
wp_weblog_title: "DemoSite"
```

## Installation and usage
Assuming the demo environement has been already created by the way of before mentioned and you've also performed `git clone` this repo.

### Manually
Ensure that you are logged in to your Ansible Automation Controller before proceeding with following steps.

### Create credentials
At leaset the following two credentails need to be defined.

#### Credential for AWS
1. Click `Credentials` in the left menu.
2. Click `Add` button.
3. Enter the following fields:
   - Name: `aws_cred`
   - Credential Type: `Amazon Web Services`
   - Access Key: your AWS_ACCESS_KEY_ID
   - Secret Key: your AWS_SECRET_ACCESS_KEY
4. Click `Save` button.

Please refer to [Ansible Doc](https://docs.ansible.com/automation-controller/4.5/html/userguide/credentials.html#amazon-web-services) for more details.

#### Credential for ssh to AWS instances
1. Click `Credentials` in the left menu.
2. Click `Add` button.
3. Enter the following fields:
   - Name: `aws_key`
   - Credential Type: `Machine`
   - SSH Private Key: your AWS private key
4. Click `Save` button.

Please refer to [Ansible Doc](https://docs.ansible.com/automation-controller/4.5/html/userguide/credentials.html#machine) for more details.

### Create inventories
1. Click `Inventories` in the left menu.
2. Click `Add` button and select `Add inventory`.
3. Enter the following fields:
   - Name: `RHEL_Demo`
4. Click `Save` button and then select `Sources` tab.
5. Click `Add` button.
6. Enter the following fields:
   - Name: `AWS`
   - Source: `Amazon EC2`
   - Credential: `aws_cred`
   - Update options: `Overwrite`, `Overwrite variables`, `Update on launch`
   - Source variables:
   ```
   ---
   # Minimal example using environment variables
   # Fetch all hosts taged with "purpose" tag as "demo" in ap-northeast-1
   
   plugin: amazon.aws.aws_ec2
   keyed_groups:
     - prefix: tag
       key: tags

   # Change regions corresponding to your environment
   regions:
     - ap-northeast-1 # adjust with your preference

   # Filter only objects taged with "purpose" tag as "demo"
   filters:
     tag:purpose: demo

   # Ignores 403 errors rather than failing
   strict_permissions: false
   ```
7. Click `Save` button.


### Create a project
1. Click `Projects` in the left menu.
2. Click `Add` button.
3. Enter the following fields:
   - Name: `RHEL_Lifecycle_Demo`
   - Organization: `Default` (or your prefered organization)
   - Execution Environment: `Default execution environment`
   - Source Control Type: `Git`
   - Source Control URL: your git repositoriy
4. Click `Save` button

Please refer to [Ansible Doc](https://docs.ansible.com/automation-controller/4.5/html/userguide/projects.html#add-a-new-project) for more details.

NOTE: You need to configure Red Hat automation hub as your primary source of content. To configure automation hub, you must create a credential and add it to the Organization’s Galaxy Credentials field (in this case "Default"). With automation hub, you have access to certified, supported collection i.e., "redhat.satellite".

### Create job templates
Each job template is equivalent to a playbook in this repository. Repeat these steps for each template/playbook that you want to use and change the variables specific to the individual playbook. Please refer to [Ansible Doc](https://docs.ansible.com/automation-controller/4.5/html/userguide/job_templates.html) for more details.

1. Click `Templates` in the left menu.
2. Click `Add` button and select `Add job template`.
3. Follow the next steps respectively.
4. Click `Save` button

#### Content View Publish
- Name: `Content View Publish`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `publish_cv.yml`

#### Promote To Dev
- Name: `Promote To Dev`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `promote_cv.yml`
- Variables:
```
---
current_lce: Library
target_lce: Development
```

#### Backup Dev VM
- Name: `Backup Dev VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `snapshot_vm.yml`
- Credentials: `aws_cred`
- Variables:
```
---
managed_vms_environment: dev
```

#### Update Dev VM
- Name: `Update Dev VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `update_vm.yml`
- Credentials: `aws_key`
- Variables:
```
---
target_hosts: tag_environment_dev
```

#### Restart Dev VM
- Name: `Restart Dev VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `restart_vm.yml`
- Credentials: `aws_cred`
- Variables:
```
---
managed_vms_environment: dev
```

#### Test Dev VM
- Name: `Test Dev VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `test_vm.yml`
- Credentials: `aws_key`
- Variables:
```
---
target_hosts: tag_environment_dev
```

#### Test Dev App
- Name: `Test Dev App`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `test_app.yml`
- Credentials: `aws_cred`
- Variables:
```
---
managed_vms_environment: dev
```

#### Promote To Prod
- Name: `Promote To Prod`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `promote_cv.yml`
- Variables:
```
---
current_lce: Development
target_lce: Production
```

#### Backup Prod VM
- Name: `Backup Prod VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `snapshot_vm.yml`
- Credentials: `aws_cred`
- Variables:
```
---
managed_vms_environment: prod
```

#### Update Prod VM
- Name: `Update Prod VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `update_vm.yml`
- Credentials: `aws_key`
- Variables:
```
---
target_hosts: tag_environment_prod
```

#### Restart Prod VM
- Name: `Restart Prod VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `restart_vm.yml`
- Credentials: `aws_cred`
- Variables:
```
---
managed_vms_environment: prod
```

#### Test Prod VM
- Name: `Test Prod VM`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `test_vm.yml`
- Credentials: `aws_key`
- Variables:
```
---
target_hosts: tag_environment_prod
```

#### Test Prod App
- Name: `Test Prod App`
- Job Type: `Run`
- Inventory: `RHEL_Demo`
- Project:  `RHEL_Lifecycle_Demo`
- Playbook: `test_app.yml`
- Credentials: `aws_cred`
- Variables:
```
---
managed_vms_environment: prod
```

### Create workflow templates
Above job templates are acutually configured as separate workflow templates for Development and for Production respectively.  Follow the next steps for each environment. Please refer to [Ansible Doc](https://docs.ansible.com/automation-controller/4.5/html/userguide/workflow_templates.html) for more details.

#### Update Dev Environment

1. Click `Templates` in the left menu.
2. Click `Add` button and select `Add workflow template`.
3. Click `Save` button.
4. Click `Start` and launch Visualizer.
5. Configure the workflow template as follows:
<img width="1506" alt="スクリーンショット 2023-11-05 15 20 39" src="https://github.com/yukshimizu/managing-rhel-lifecycle-demo/assets/24378327/4f5519b6-67f3-4454-9fad-45198802ce55">

6. Click `Save` button.
7. Click `Survery` tab and click `Add` button.
8. Add the following two surveys and enable them:
    - ErrataByDate
        - Question: Enter ErrataByDate
        - Answer variable name: `satellite_cv_end_date`
        - Answer Type: Text
    - Satellite Password
        - Questoin: Enter Satellite Password
        - Answer variable name: `satellite_admin_passwd`
        - Answer Type: Password

NOTE: Although `satellite_admin_passwd` should be encrypted in production, using vault for example, I just use easier way for demo purpose.

#### Update Prod Environment

1. Click `Templates` in the left menu.
2. Click `Add` button and select `Add workflow template`.
3. Click `Save` button.
4. Click `Start` and launch Visualizer.
5. Configure the workflow template as follows:
<img width="1499" alt="スクリーンショット 2023-11-05 15 21 00" src="https://github.com/yukshimizu/managing-rhel-lifecycle-demo/assets/24378327/19121ae5-ea09-43b1-9c86-4ce13bdec4bf">

6. Click `Save` button.
7. Click `Survery` tab and click `Add` button.
8. Add the following two surveys and enable them:
    - ErrataByDate
        - Question: Enter ErrataByDate
        - Answer variable name: `satellite_cv_end_date`
        - Answer Type: Text
    - Satellite Password
        - Questoin: Enter Satellite Password
        - Answer variable name: `satellite_admin_passwd`
        - Answer Type: Password

NOTE: When running the workflow for Production , `satellite_cv_end_date` needs to be set identical to the input for Development.
