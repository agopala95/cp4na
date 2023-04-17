# cp4na

Repo for OCP/ODF/CP4NA install associated artifacts

## Workflow for installing a SNO cluster using IBM CP4NA automation scripts

## DEPLOY SNO USING IBM CP4NA + AUTOMATION SCRIPTS STEPS

1. Install the RHACM operator and multiclusterhub instance is installed and enable/deploy the instance’s corresponding assisted installer service Central Infrastructure Management Service(CIM).
__Ansible files to operate:__
	__RHACM operator__
https://github.com/redhat-eets/cp4na/blob/main/ansible/acm_operator_install.yaml
__RHACM multiclusterhub instance__
https://github.com/redhat-eets/cp4na/blob/main/ansible/acm_multiclusterhub.yaml
__CIM__
https://github.com/redhat-eets/cp4na/blob/main/ansible/acm_postinstall.yaml

2. Install the OpenShift GitOps (ArgoCD) operator
__Ansible files to operate:__
__OpenShift GitOps operator__
https://github.com/redhat-eets/cp4na/blob/main/ansible/openshift_gitops_install.yaml

3. Install the IBM CP4NA operator, followed by its instance while ensuring that the SitePlanner component is enabled as well.
 __Ansible files to operate:__
__Install CP4NA operator__
https://github.com/redhat-eets/cp4na/blob/main/ansible/cp4na_operator_install.yaml
__Deploy CP4NA instance with SitePlanner component.__
https://github.com/redhat-eets/cp4na/blob/main/ansible/cp4na_instance.yaml
__Change the Install Plan approval method of the CP4NA instance to “Manual”__
https://github.com/redhat-eets/cp4na/blob/main/ansible/cp4na_subscription_edit.yaml

	__Note:__ Add the following routes to the file /etc/hosts while mapping them to the hub cluster’s ingress VIP:
	__CP4NA Ishtar Route__
	```sh
	oc get route cp4na-o-ishtar -n ibmcp4na -o jsonpath='{.spec.host}'
	```
	__CP4NA Instance UI Route__
	```sh
	oc get orchestration tnco -o 'jsonpath={..status.uiendpoints.orchestration}' -n ibmcp4na
	```
4. Install the necessary prerequisites needed for the CP4NA’s siteplanner component to deploy OCP SNO with RHACM and ArgoCD in conjunction. The following installations are taken care of by the ansible playbook linked below:
	- Python packages through pip [Kubernetes, OpenShift, PyYAML]
	- Helm
	```
	oc client binary
	```
	- ArgoCD cli

	__Ansible files to operate:__
https://github.com/redhat-eets/cp4na/blob/main/ansible/pre_req.yaml

5. Install the ansible lifecycle driver for CP4NA operator with the help of helm and onboard it as per the link here manually or use the Ansible playbook given below

	https://github.com/redhat-eets/cp4nadoc/blob/master/prereqs/ansibleDriverDeployment.md
Reference - https://github.com/IBM/ansible-lifecycle-driver/blob/master/docs/install_with_helm.md

	__Ansible files to operate:__
https://github.com/redhat-eets/cp4na/blob/main/ansible/cp4na_ansible_driver_install.yaml

	__To do it manually:__
wget https://github.com/IBM/ansible-lifecycle-driver/releases/download/<version>/ansiblelifecycledriver-<version>.tgz

	Version - 3.5.1

	wget https://github.com/IBM/ansible-lifecycle-driver/releases/download/3.5.1/ansiblelifecycledriver-3.5.1.tgz

	Change to the CP4NA namespace
```
oc project ibmcp4na
```
kafka host value must be set as follows, in values.yaml file of the helm package, depending on the CP4NA versions:

	For pre CP4NA v2.3, the kafka host must be iaf-system-kafka-bootstrap
For CP4NA v2.3/v2.3+, the kafka host must be cp4na-o-events-kafka-bootstrap

	Version - 3.5.1
```
	helm install ansiblelifecycledriver-3.5.1.tgz --generate-name --setapp.config.override.messaging.connection_address=iaf-system-kafka-bootstrap:9092
```
	Extract the TLS certificate for the driver to onboard ansible driver as resource driver to CP4NA
```
oc get secret ald-tls -o 'go-template={{index .data "tls.crt"}}' | base64 -d > ald-tls.pem
```

	The ansible lifecycle driver pod should be visible in the namespace where CP4NA operator is installed.

6. Get the password for the default “admin” user of the CP4NA GUI by using the following command and then set access roles for the admin user in the GUI by first logging in with the obtained credentials as shown in the screenshots below.

	Get admin password
```
oc get secret platform-auth-idp-credentials -n ibm-common-services -o yaml echo TzUySlV2bEl6Unk3U3JudUgzU3VrbXFnUHRJb0VNNEI= | base64 -d
```
	Set access roles through the CP4NA GUI

	Command to get the CP4NA GUI url link
```
oc get routes cpd -o jsonpath='{.spec.host}' -n ibmcp4na
```
# INSERT IMAGES HERE

	Automation Administrator, SLMAdmin and SPAdmin are the roles to be chosen.
7. Next, generate and take note of the API key for the user “admin” as seen in the screenshots below.
# INSERT IMAGES HERE

8. Install the tool lmctl to onboard the Ansible Lifecycle Driver to the CP4NA operator and download the assembly descriptors that contain automation scripts for CP4NA to deploy SNO.

	***Ansible files to operate:***
https://github.com/redhat-eets/cp4na/blob/main/ansible/cp4na_lmctl_deploy.yaml

	To do it manually:-
Create a config yaml file for lmctl and make it point to the CP4NA setup and export the path.
```
[openshift@dhcpserver ~]$ vi lmctl.yaml
```
```yaml
active_environment: dev
environments:
  dev:
    tnco:
      address: https://cp4na-o-ishtar-ibmcp4na.apps.ocphub4ztp.cp4na.bos2.lab
      api_key: IWlzXggfPdHL495uo25kqeCAZlODnjdFcbI0JROW
      auth_address: https://cpd-ibmcp4na.apps.ocphub4ztp.cp4na.bos2.lab/icp4d-api/v1/authorize
      auth_mode: zen
      secure: true
      username: admin
```
The api_key and username were obtained from the previous step.
The address can be obtained from the following command:
```
oc get route cp4na-o-ishtar -o jsonpath='{.spec.host}'
```
```yaml
auth_address: CP4NA SiteplannerGUI url + icp4d-api/v1/authorize
```
Next, export the file path as LMCONFIG
```sh
export LMCONFIG=/home/openshift/lmctl.yaml
```
Install lmctl using the link - https://github.com/IBM/lmctl/blob/master/docs/getting-started.md
```sh
python3 -m pip install lmctl
```
Add resource driver through lmctl
```sh
	lmctl resourcedriver add --type ansible --url https://ansible-lifecycle-driver:8293 dev --certificate ald-tls.pem
```
	The descriptors of interest are : ocp-cluster-automation-develop, server-automation-master, prereqs-datamodel-master, nodeconfig-master

	__Note:__ Files having commented out lines or appended lines

	ocp-cluster-automation-develop/Contains/cluster/Contains/predeploy/Lifecycle/ansible/scripts/configure_oc.yml

	prereqs-datamodel-master/Contains/ddi/Contains/deploy/Lifecycle/ansible/scripts/Install.yaml
```yaml
   vars:
     log_timestamp: "{{ lookup('pipe', 'date +%Y%m%d-%H-%M-%S') }}"
    ansible_become_pass: "{{  ddi.config_context.password }}"     
```
 add this line
```yaml
sno_node: {}
name: "sno"
```
* Git clone the repos as seen in the link
https://github.com/redhat-eets/cp4na_assembly_scripts
* Go to each repo folder and run the command:
```sh
lmctl project push dev
```
Example
```sh
[openshift@dhcpserver ~]$ cd Downloads/ocp-cluster-automation-develop/
[openshift@dhcpserver ocp-cluster-automation-develop]$ lmctl project push dev
```
9. In this step, the SitePlanner needs data to be collected and filled before SNO deployment. The data will include site information like node, provisioner, jumphost, dns-dhcp details, etc.

	__Note:__
Please ensure the following steps are done before putting data into the SitePlanner
	* Create assisted-deployment-pull-secret in open-cluster-management namespace (This will be created as part of executing the playbook at https://github.com/redhat-eets/cp4na/blob/main/ansible acm_postinstall.yaml)
	* Add ZTP github repo in argocd
	* Add host entries in /etc/hosts

	To do it manually:
Follow steps from the official document to model the siteplanner with SNO site data and management cluster data and make sure Site device has a build option.

	After generating API key for “admin” user in step 6, use the following command listed here
	https://www.ibm.com/docs/en/cloud-paks/cp-network-auto/2.2.x?topic=apis-authenticating-rest-api-requests
	to get an access token which is used to access the APIs.
```sh
curl -s -k -X POST https://cpd-ibmcp4na.apps.ocphub4ztp.cp4na.bos2.lab/icp4d-api/v1/authorize -H 'Content-Type: application/json' -d '{ "username": "admin", "api_key": "IWlzXggfPdHL495uo25kqeCAZlODnjdFcbI0JROW"  }' | python -c 'import sys, json; print json.load(sys.stdin)["access_token"]'
```
Data modeling order for site planner:
		* Create Manufacturer [Dell]
		* Create Device Type [PowerEdge 650]
		* Create Device Roles  [DHCP-DNS, Git, Provisioner, OCP-Node servers]
		* Create Site [Management, Billerica]
		* Create Devices [DHCP-DNS, Git, Provisioner, OCP-Node servers]
		* Create Config Context [Image Registry, Management Cluster Config, SNO cluster Config]
		* Create Interfaces for ocp-node [eno12399, ipmi_interface]
		* Create Service for ipmi_interface
		* Create Automation Contexts

		To do it through SitePlanner REST APIs:-
On gaining access to the REST API collection compiled by Postman client, the following REST API calls can be used to do data modeling.

	REST API ACCESS
		* Using the API key generated for the admin user in step 6, the user can run the GenerateAccessToken API call to get an access token used to authorize the API calls being made to model data into the SitePlanner.

		INSERT IMAGE HERE
		*Once the access token is obtained, the first step is to create a manufacturer (the BMC type of the lab machine intended to be used for SNO) and this can be done by executing the CreateManufacturer API call.

		* In the request body, the name has to be a recognized predefined value from the set - Dell, SuperMicro, HPE, Lenovo.
		* The slug value can be the same as the name value too as it is virtually a shortened name or nickname (as is with the case with the following API calls).
		* Next, the user can create the device type which is the model of the manufacturer being used. This can be done through the CreateDeviceType API request.

		Body:- (Example is that of Dell and its sample model)
```yaml
{
  "manufacturer": { "name": "Dell", "slug": "dell"},
  "model": "PowerEdge R650",
  "slug": "poweredge-r650",
  "u_height": 1,
  "is_full_depth": true
}
```
		* The user can now proceed to label the device roles at play for each server/lab machine that’s involved (which can include DNS/DHCP, provisioner, GIT server, SNO OCP node). This can be done by executing multiple API calls through the CreateDeviceRole request for each device role.
		* The above example shows an execution to create a device role for the provisioning host or server.
		* The color field has a RGB value that sets the color associated with the device role for easier viewing in the UI. The vm_role field can be set to false.

		__Note:__ Use hyphens instead of underscore for name fields in both device roles and devices

		* For the provisioner role, the name field is provisioner.
		* Similarly, device roles can be set for the DNS/DHCP server where the name field in particular has to be DHCP DNS Server.
		* For the git server role, it is Git Server.
		* For the SNO ocp node role, it can be ocp-node.

		* In this step, the sites can be created under which the devices to be created as part of step f are placed into. For each site, the user can make an API request through CreateSite.
		* Two sites have to created - Management (Site which contains the DHCP/DNS device, Git server device, provisioner device), Billerica (Site which contains the SNO ocp node device)

		* Body for Management site:
```yaml
{
   "name": "Management",
   "slug": "management",
   "status": "active",
   "region": null,
   "tenant": null,
   "facility": "",
   "asn": null,
   "time_zone": "US/Eastern",   [not accepting null]
   "description": "",
   "physical_address": "",
   "shipping_address": "",
   "latitude": null,
   "longitude": null,
   "contact_name": "",
   "contact_phone": "",
   "contact_email": "",
   "comments": "",
   "tags": [],
   "custom_fields": {}
}
```
		* Body for Billerica site:
```yaml
{
   "name": "Billerica",
   "slug": "billerica",
   "status": "active",
   "region": null,
   "tenant": null,
   "facility": "",
   "asn": null,
   "time_zone": "US/Eastern",
   "description": "",
   "physical_address": "",
   "shipping_address": "",
   "latitude": null,
   "longitude": null,
   "contact_name": "",
   "contact_phone": "",
   "contact_email": "",
   "comments": "",
   "tags": [
       "du",
       "skip_server-automation",
       "skip_ddi_automation",
       "skip_node_config"
   ],
   "custom_fields": {}
}
```
		* Body for Showcase site:
```yaml
{
   "name": "Showcase",
   "slug": "showcase",
   "status": "active",
   "region": null,
   "tenant": null,
   "facility": "",
   "asn": null,
   "time_zone": "US/Eastern",
   "description": "",
   "physical_address": "",
   "shipping_address": "",
   "latitude": null,
   "longitude": null,
   "contact_name": "",
   "contact_phone": "",
   "contact_email": "",
   "comments": "",
   "tags": [
       1,2,3,4
   ],
   "custom_fields": {}
}
```
		* Once the device roles are created, the actual devices with these roles can be now created. This can be done by using the CreateDevice API request each time a device has to be created.
		* The devices to be created with their respective name field values are - DNS DHCP Server, Git Server, provisioner and ocp-node.
__Note:__ In the request body, the device is linked to a site created earlier through the site field.
Management site - DNS DHCP Server, Git Server, provisioner
		* Billerica site - ocp-node

		Body for DNS DHCP Server:
```yaml
{
 "name": "DNS DHCP Server",
 "device_type": {
     "manufacturer":{
         "name": "Dell"
     },
     "model":"PowerEdge R650"
 },
 "device_role": {
     "name": "DHCP DNS Server"
 },
 "site": {
     "name": "Management"
 },
 "status": "active",
 "local_context_data": {
       "host": "192.168.19.11",
       "user": "openshift",
       "dns_file": "/etc/hosts",
       "password": "openshift",
       "dhcp_file": "/etc/dnsmasq/dnsmasq.conf"
 },
     "config_context": {
       "host": "192.168.19.11",
       "user": "openshift",
       "dns_file": "/etc/hosts",
       "password": "openshift",
       "dhcp_file": "/etc/dnsmasq/dnsmasq.conf"
   }
}
```
The user can note that the body has the config context field with key/value fields within it which includes the values as follows:
```
host: IP address of the server hosting the DNS/DHCP service
user: server login username
dns_file: file to be referred to resolve hostnames
password: server login password
dhcp_file: file to be referred to check the DHCP configuration
```
The local_context_data can contain the same fields as that of the config_context field

		Body for Git Server:
```yaml
{
 "name": "Git Server",
 "device_type": {
     "manufacturer":{
         "name": "Dell"
     },
     "model":"PowerEdge R650"
 },
 "device_role": {
     "name": "Git Server"
 },
 "site": {
     "name": "Management"
 },
 "status": "active",
 "local_context_data": {
       "git_repo": "https://github.com/agopala95/test-sno-ztp.git",
       "git_token": "<git token>",
       "git_branch": "test_ztp",
       "git_username": "agopala95"
 },
     "config_context": {
       "git_repo": "https://github.com/agopala95/test-sno-ztp.git",
       "git_token": "<git token>",
       "git_branch": "test_ztp",
       "git_username": "agopala95"
   }
}
```
The user can note that the body has the config context field with key/value fields within it which includes the values as follows:
```yaml
git_repo: HTTPS URL link to the github repo hosting the SNO OCP SiteConfig and related manifest files
git_token: Github account personal access token
git_branch: Branch of the repo to be linked
git_username: Github account username
```
The local_context_data can contain the same fields as that of the config_context field.

		Body for provisioner:
```yaml
{
 "name": "provisioner",
 "device_type": {
     "manufacturer":{
         "name": "Dell"
     },
     "model":"PowerEdge R650"
 },
 "device_role": {
     "name": "provisioner"
 },
 "site": {
     "name": "Management"
 },
 "status": "active",
 "local_context_data": {
       "host": "192.168.19.11",
       "user": "openshift",
       "password": "openshift"
 },
     "config_context": {
       "host": "192.168.19.11",
       "user": "openshift",
       "password": "openshift"
   }
}
```
	The user can note that the body has the config context field with key/value fields within it which includes the values as follows:
```yaml
host: IP address of the server acting as the provisioning host.
user: server login username
password: server login password
```
The local_context_data can contain the same fields as that of the config_context field.

		Body for ocp-node:
```yaml
{
 "name": "ocp-node",
 "device_type": {
     "manufacturer":{
         "name": "Dell"
     },
     "model":"PowerEdge R650"
 },
 "device_role": {
     "name": "ocp-node"
 },
 "site": {
     "name": "Billerica"
 },
 "status": "active",
 "local_context_data": {
       "ipmi_userid": "root",
       "ipmi_password": "calvin"
 },
  "tags": [
       "du",
       "skip_bios_automation",
       "skip_storage_automation",
       "skip_firmware_automation"
   ],
     "config_context": {
       "ipmi_userid": "root",
       "ipmi_password": "calvin"
   }
}
```
The user can note that the body has the config context field with key/value fields within it which includes the values as follows:
```yaml
ipmi_userid: BMC/IPMI login username
ipmi_password: BMC/IPMI login password
```
The local_context_data can contain the same fields as that of the config_context field.

		Configuration data needed for the SNO node like the hub/management cluster data and Image registry data can be compiled into a “config context”.

		Three config contexts are created for the use case here: [Image Registry Configuration, Management Cluster Configuration, SNO cluster properties]

		The API requests for each of these config contexts can be made using the CreateConfigContext request.

		__Note:__ The Billerica site has to be linked to each of the config contexts. This can be done by mapping the Site ID generated as part of the CreateSite API request.

		Body for Image Registry Configuration
```yaml
{
 "name": "Image Registry Configuration",
 "weight": 1000,
 "description": "",
 "is_active": true,
  "sites": [ 6 ],
 "data": {
       "management_cluster_config": {
           "imageregistry": {
               "4.9": {
                   "server": "quay.io",
                   "ocp_repo": "openshift-release-dev/ocp-release",
                   "ztp_image": {
                       "repo": "openshift4/ztp-site-generate-rhel8",
                       "server": "registry.redhat.io",
                       "version": "v4.9.0"
                   },
                   "image_repo": "ocp4/openshift4",
                   "ocp_minor_version": "4.9.32"
               },
               "cu_du_registry": {
                   "auth": "reg-access-sa:eyJhbGciOiJSUzI1NiIsImtpZCI6Imt0bVgzTzZXcHVVZWg3NHU0ZGgxUWlKWjVLbWNwY0RhWXR1ZEtKNjNMT1EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtdGNpbCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJyZWctYWNjZXNzLXNhLXRva2VuLTRtODg4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InJlZy1hY2Nlc3Mtc2EiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlMzdlMjc1Zi1hZGZiLTQ2MWUtYTBjMi01MDg2MjlkMTA1NDciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6bXRjaWw6cmVnLWFjY2Vzcy1zYSJ9.Hufd27U5SnSuxJnMv2_MY1Rrb1Hd78SEhIJOnCLfTDYxi1gjfWAQP400TT2yTA15ClVCeluVSEvaq77n0IvZ3wYGNVHVH5TpSyuKvjMntq36GoliDYG37ReQ01tg6ExRGqzyrTOasEK15ygtEz_nkBAL1y0UMRZBrUj6z4Eric09-VHnbBrgRtS14JnR0UBiotv18aCHmNq3_BCJoiQhXbiRK5BeXY6xnHijLR6Q1SbugWvTQ1zSpOyjZAVz6dIU6xd9tWpdOt2e_nCqhNQ22PS-LOh5RhM_nt7-hCiK5u8YByCiMRety-uqgoRwmnPyDhYHpdg6YVBEGyfrskzRjwW6TwqNu-P3a9sCBY8LMj6Y775to46lG2DhfWDe_nEs8eU7_Df-FLjBqx-8jGPfEYuZS5Sf-Gikw6Peld5-jKjlOdhYfFsBcgi5seNRx7cUS7BaOke5mNvdRi_D5kdKx39z0EWebPg3M4dl3rOT-R0LGU0ZsOPDYaFgqoRMlys_Y8rcYKk7RkyMaryiVHN0Orgrmai4wMw_Xb6D-jLP5wJFSeI2kKAr_ovbsilntc9iqaEWxSRz0GMjDvC4vRwufO1bQ0OwzobvBqC9OY9O16x2CndPG0zLM2vSqHFT8DSHAjxxuJdy2V-ap1O9pKh2GrSTxcd08rmL3bfjtq4KRs4",
                   "server": "default-route-openshift-image-registry.apps.ocp.lab.com",
                   "pull_secret": {
                       "auths": {
                           "default-route-openshift-image-registry.apps.ocp.lab.com": {
                               "auth": "cmVnLWFjY2Vzcy1zYTpleUpoYkdjaU9pSlNVekkxTmlJc0ltdHBaQ0k2SW10MGJWZ3pUelpYY0hWVlpXZzNOSFUwWkdneFVXbEtXalZMYldOd1kwUmhXWFIxWkV0S05qTk1UMUVpZlEuZXlKcGMzTWlPaUpyZFdKbGNtNWxkR1Z6TDNObGNuWnBZMlZoWTJOdmRXNTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5dVlXMWxjM0JoWTJVaU9pSnRkR05wYkNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZqY21WMExtNWhiV1VpT2lKeVpXY3RZV05qWlhOekxYTmhMWFJ2YTJWdUxUUnRPRGc0SWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXpaWEoyYVdObExXRmpZMjkxYm5RdWJtRnRaU0k2SW5KbFp5MWhZMk5sYzNNdGMyRWlMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObGNuWnBZMlV0WVdOamIzVnVkQzUxYVdRaU9pSmxNemRsTWpjMVppMWhaR1ppTFRRMk1XVXRZVEJqTWkwMU1EZzJNamxrTVRBMU5EY2lMQ0p6ZFdJaU9pSnplWE4wWlcwNmMyVnlkbWxqWldGalkyOTFiblE2YlhSamFXdzZjbVZuTFdGalkyVnpjeTF6WVNKOS5IdWZkMjdVNVNuU3V4Sm5NdjJfTVkxUnJiMUhkNzhTRWhJSk9uQ0xmVERZeGkxZ2pmV0FRUDQwMFRUMnlUQTE1Q2xWQ2VsdVZTRXZhcTc3bjBJdlozd1lHTlZIVkg1VHBTeXVLdmpNbnRxMzZHb2xpRFlHMzdSZVEwMXRnNkV4UkdxenlyVE9hc0VLMTV5Z3RFel9ua0JBTDF5MFVNUlpCclVqNno0RXJpYzA5LVZIbmJCcmdSdFMxNEpuUjBVQmlvdHYxOGFDSG1OcTNfQkNKb2lRaFhiaVJLNUJlWFk2eG5IaWpMUjZRMVNidWdXdlRRMXpTcE95alpBVno2ZElVNnhkOXRXcGRPdDJlX25DcWhOUTIyUFMtTE9oNVJoTV9udDctaENpSzV1OFlCeUNpTVJldHktdXFnb1J3bW5QeURoWUhwZGc2WVZCRUd5ZnJza3pSandXNlR3cU51LVAzYTlzQ0JZOExNajZZNzc1dG80NmxHMkRoZldEZV9uRXM4ZVU3X0RmLUZMakJxeC04akdQZkVZdVpTNVNmLUdpa3c2UGVsZDUtaktqbE9kaFlmRnNCY2dpNXNlTlJ4N2NVUzdCYU9rZTVtTnZkUmlfRDVrZEt4Mzl6MEVXZWJQZzNNNGRsM3JPVC1SMExHVTBac09QRFlhRmdxb1JNbHlzX1k4cmNZS2s3Umt5TWFyeWlWSE4wT3Jncm1haTR3TXdfWGI2RC1qTFA1d0pGU2VJMmtLQXJfb3Zic2lsbnRjOWlxYUVXeFNSejBHTWpEdkM0dlJ3dWZPMWJRME93em9idkJxQzlPWTlPMTZ4MkNuZFBHMHpMTTJ2U3FIRlQ4RFNIQWp4eHVKZHkyVi1hcDFPOXBLaDJHclNUeGNkMDhybUwzYmZqdHE0S1JzNA==",
                               "user": "reg-access-sa",
                               "email": "pmallana@in.ibm.com",
                               "password": "eyJhbGciOiJSUzI1NiIsImtpZCI6Imt0bVgzTzZXcHVVZWg3NHU0ZGgxUWlKWjVLbWNwY0RhWXR1ZEtKNjNMT1EifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJtdGNpbCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJyZWctYWNjZXNzLXNhLXRva2VuLTRtODg4Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InJlZy1hY2Nlc3Mtc2EiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJlMzdlMjc1Zi1hZGZiLTQ2MWUtYTBjMi01MDg2MjlkMTA1NDciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6bXRjaWw6cmVnLWFjY2Vzcy1zYSJ9.Hufd27U5SnSuxJnMv2_MY1Rrb1Hd78SEhIJOnCLfTDYxi1gjfWAQP400TT2yTA15ClVCeluVSEvaq77n0IvZ3wYGNVHVH5TpSyuKvjMntq36GoliDYG37ReQ01tg6ExRGqzyrTOasEK15ygtEz_nkBAL1y0UMRZBrUj6z4Eric09-VHnbBrgRtS14JnR0UBiotv18aCHmNq3_BCJoiQhXbiRK5BeXY6xnHijLR6Q1SbugWvTQ1zSpOyjZAVz6dIU6xd9tWpdOt2e_nCqhNQ22PS-LOh5RhM_nt7-hCiK5u8YByCiMRety-uqgoRwmnPyDhYHpdg6YVBEGyfrskzRjwW6TwqNu-P3a9sCBY8LMj6Y775to46lG2DhfWDe_nEs8eU7_Df-FLjBqx-8jGPfEYuZS5Sf-Gikw6Peld5-jKjlOdhYfFsBcgi5seNRx7cUS7BaOke5mNvdRi_D5kdKx39z0EWebPg3M4dl3rOT-R0LGU0ZsOPDYaFgqoRMlys_Y8rcYKk7RkyMaryiVHN0Orgrmai4wMw_Xb6D-jLP5wJFSeI2kKAr_ovbsilntc9iqaEWxSRz0GMjDvC4vRwufO1bQ0OwzobvBqC9OY9O16x2CndPG0zLM2vSqHFT8DSHAjxxuJdy2V-ap1O9pKh2GrSTxcd08rmL3bfjtq4KRs4"
                           }
                       }
                   }
               }
           }
       }
 }
}
```
Body for Management Cluster Configuration
```yaml
{
 "name": "Management Cluster Configuration",
 "weight": 1000,
 "description": "",
 "is_active": true,
  "sites": [ 6 ],
 "data": {
       "management_cluster_config": {
           "cp4na": {
               "api_key": "IWlzXggfPdHL495uo25kqeCAZlODnjdFcbI0JROW",
               "ocp_user": "kubeadmin",
               "username": "admin",
               "ocp_server": "api.ocphub4ztp.cp4na.bos2.lab:6443",
               "ocp_password": "d4wAN-gQFqb-ZqKLs-YDaqi",
               "tnco_namespace": "ibmcp4na"
           },
           "rhacm": {
               "user": "kubeadmin",
               "server": "api.ocphub4ztp.cp4na.bos2.lab:6443",
               "password": "d4wAN-gQFqb-ZqKLs-YDaqi"
           },
           "argocd": {
               "url": "openshift-gitops-server-openshift-gitops.apps.ocphub4ztp.cp4na.bos2.lab",
               "namespace": "openshift-gitops",
               "route_name": "openshift-gitops-server",
               "secret_name": "openshift-gitops-cluster"
           },
           "cluster": {
               "url": "https://api.ocphub4ztp.cp4na.bos2.lab:6443",
               "cert": "",
               "name": "ocphub4ztp",
               "pass": "d4wAN-gQFqb-ZqKLs-YDaqi",
               "user": "kubeadmin",
               "domain": "cp4na.bos2.lab"
           },
           "max_timeout": 100
       }
 }
}
```
Body for SNO cluster properties
```yaml
{
 "name": "SNO cluster properties",
 "weight": 1000,
 "description": "",
 "is_active": true,
  "sites": [ 6 ],
 "data": {
       "cpuset": "0-3,12-15",
       "cluster_config": {
           "profile": "du",
           "cluster_type": "sno",
           "cluster_domain": "cp4na.bos2.lab",
           "cluster_version": 4.9,
           "airgap_environment": false
       },
       "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDsCHghGt5v9XtXgKSeiIUurcUV5VIIv8TTYwow3RIuGJRASxqjWXW6vaHA1cnjOtwZUaylOJryLDTlzP91Uy1+rsjO7ZHxrJw2/tPzoFLDd8uBoqzQESz+Y8yq5RKJ/osRsH4/r1gVA2efrAOcvXpozg4P1mUJnOuhhcslfrESqJVz2JgvDMtq8uVmG0nH/yyquwgvvpD1MMcixWnS89qKnIPRV+zcBZ3soIzC6ikGBLh8B+JnhXdhYzF8xqYrDy5VaJn3P4NOGy67ebwiYQJ0y3+c6whW52q19GIpGK7gRHj5NEWUUSd6t69LyM/mSsx+ubDqLqXu+uzqrnL9yDjLpnX2FxiyapkaGmD4siqgJtUwngEaCpqo/L5ZLJtFmBmc77dAVU271r9TDI+1/angZv+7QYYIvfC2c7DCd6QMzKA5DOzM9it230z+YvE3AuXys2RDZuRjmxODSTfdBXnXCTq6Pi43eEAd+YLFkhmxkQvwCELUVLZZQAZlhM6158k= openshift@dhcpserver",
       "rootdevicehints_devicename": "/dev/sda"
 }
}
```
		* Next, the IPMI interface from the chosen lab server to be linked for the SNO deployment can be created through the CreateInterface API request.
		* The device to be linked here is the SNO node device - ocp-node

		Body for interface ipmi_interface:
```yaml
{
 "device": {
     "name":"ocp-node"
 },
 "name": "ipmi_interface",
 "type": "10gbase-t",
 "enabled": true,
 "mgmt_only": false
}
```
	* The BMC IP address is linked to the ipmi_interface interface by executing the AssignIPAddress API request.

	Body:
```yaml
{
"family": 4,
 "address": "192.168.19.159/24",
 "status": "active",
 "interface": {
       "device": {
           "name": "ocp-node"
       },
       "name": "ipmi_interface"
 }
}
```
Similarly, the same set of steps can be taken to create interface data for the boot interface of the SNO node which is eno12399 for the use case here in the Billerica lab

	Body of the CreateInterface API request for interface eno12399:
```yaml
{
 "device": {
     "name":"ocp-node"
 },
 "name": "eno12399",
 "type": "10gbase-t",
 "enabled": true,
 "lag": null,
 "mtu": 1500,
 "mac_address": "B4:96:91:EC:31:EC",
 "mgmt_only": false,
 "description": ""
}
```
Body of the AssignIPAddress API request for interface eno12399:
```yaml
{
"family": 4,
 "address": "192.168.19.90/25",
 "status": "active",
 "interface": {
       "device": {
           "name": "ocp-node"
       },
       "name": "eno12399"
 }
}
```
Next, service data has to be created for ipmi_interface to reach the BMC management IP address of the SNO node device ocp-node. This can be done through the CreateService API request.

	__Note:__ The ipaddresses field contains the IPaddress ID generated when the AssignIPAddress API request is executed for the IP address of the interface ipmi_interface.

	Body for the ipmi_service service:
```yaml
{
 "device": {
     "name":"ocp-node"
 },
 "name": "ipmi_service",
 "port": 443,
 "protocol": "tcp",
 "ipaddresses": [ 4 ]
}
```
The final data to be uploaded is the automation context data which contains a JQuery needed for SNO deployment. This can be done by executing the CreateAutomationContext API request.

	Body of the automation context OCP Deployment:
```yaml
{
 "name": "OCP deployment",
 "descriptorName": "assembly::ocp-automation::1.0",
 "object_type": "dcim.site",
 "data_expressions": "query ocp_cluster_config($name: String!) { \r\n    sites: sites(filters: {name: $name}) {\r\n      __key: id\r\n      name\r\n      tags   \r\n      nodes: devices(filters: {device_role__name: \"ocp-node\"}) { \r\n        config_context\r\n     }\r\n      master_nodes: devices(filters: {device_role__name: \"ocp-node\"}) {\r\n        __key: id\r\n        name\r\n        tags\r\n        config_context\r\n        device_type {\r\n          manufacturer {\r\n            name\r\n          }\r\n          model\r\n        }\r\n        site {\r\n          name\r\n          region {\r\n            name\r\n          }\r\n        }\r\n        platform {\r\n          name\r\n        }\r\n        interfaces(filters: {name: \"eno12399\"}) {\r\n          name\r\n          mac_address\r\n member_interfaces {\r\n            name\r\n            mac_address\r\n          }\r\n       tagged_vlans {\r\n            vid\r\n          }\r\n          untagged_vlan {\r\n            vid\r\n          }\r\n          ip_addresses {\r\n            address\r\n          }\r\n        }\r\n        bond_interface:interfaces(filters: {name: \"eno12399\"}) {\r\n          ip_addresses {\r\n            address\r\n          }\r\n        }\r\n        ipmi_interface:interfaces(filters: {mgmt_only: true}) {\r\n          ip_addresses {\r\n            address\r\n          }\r\n        }\r\n        services(filters: {name: \"ipmi_service\"}) {\r\n          ipaddresses {\r\n            address\r\n          }\r\n          port\r\n        }\r\n      }\r\n      worker_nodes: devices(filters: {device_role__name: \"ocp-worker-node\"}) {\r\n        __key: id\r\n        name\r\n        tags\r\n        config_context\r\n        device_type {\r\n          manufacturer {\r\n            name\r\n          }\r\n          model\r\n        }\r\n        site {\r\n          name\r\n          region {\r\n            name\r\n          }\r\n        }\r\n        platform {\r\n          name\r\n        }\r\n        interfaces(filters: {name: \"ens3f0\"}) {\r\n          name\r\n          mac_address\r\n     member_interfaces {\r\n            name\r\n            mac_address\r\n          }\r\n          ip_addresses {\r\n            address\r\n          }\r\n        }\r\n       ipmi_interface:interfaces(filters: {mgmt_only: true}) {\r\n          ip_addresses {\r\n            address\r\n           }               \r\n        }\r\n        services(filters: {name: \"ipmi_service\"}) {\r\n          ipaddresses {\r\n            address\r\n          }\r\n          port\r\n        }\r\n      }\r\n     }\r\n\r\n  provisioner: device(filters: {device_role__name: \"provisioner\"}) {\r\n    config_context\r\n  }\r\n  ddi_server: device(filters: {device_role__name: \"DHCP DNS Server\"}) {\r\n    config_context\r\n  }\r\n  git_server: device(filters: {device_role__name: \"Git Server\"}) {\r\n    config_context\r\n  }\r\n}"
}
```
The values highlighted in the request body above are the corresponding values exactly as generated earlier in the form of:
		- ocp-node device role
		- eno12399 interface
		- ipmi_interface interface
		- ipmi_service service
		- provisioner device role
		- DHCP DNS Server device role
		- Git Server device role

		Also, the object_type field value represents an automation context of Site type so the value would be dcim.site

10. Next, a deployment location has to be added through the CP4NA SitePlanner GUI.

	Go to SitePlanner UI - system → Deployment location | click Add

	Enter details for deployment location name - core
Resource Manager - brent
Infrastructure - openstack
11. Now that the required data has been modeled into the SitePlanner, SNO deployment is ready to be executed. This is done by doing a “build Site” action either through the CP4NA UI or the REST API request BuildSite.

	Body for the request:
```yaml
{
 "object_type": "dcim.site",
 "object_pk": "6",
 "dry_run": false
}
```
* The object_type field represents the Site type dcim.type
* The object_pk field represents the Site ID of Billerica that was generated earlier as part of the CreateSite API request.

—----------------------------------------------------------------------------------------------------------------------------

12. If the above steps don’t work for a complete deployment of the SNO, the following steps may need to be added before a run.

	Make sure virtual disks are created in the physical lab machines to be used as an SNO node.

	Use hyphens instead of underscore for device roles and devices
	* provisioner
	* DHCP DNS Server
	* ocp-node
	* ocp-worker-node

	Create DNS server
```yaml
{
    "dhcp_file": "/etc/dnsmasq.d/duocpdell.lab.conf",
    "dns_file": "/etc/hosts",
    "host": "10.0.10.72",
    "password": "cpna",
    "user": "cpna"
}
```
Site Billerica - tags: skip_server-automation, du, skip_ddi_automation, skip_node_config

	Device - opc-node - tags: du, skip_bios_automation, skip_storage_automation, skip_firmware_automation

	Make sure cpuset is defined in cluster config
```yaml
{
    "cluster_config": {
        "airgap_environment": false,
        "cluster_domain": "lab.com",
        "cluster_name": "duocpdell",
        "cluster_type": "sno",
        "cluster_version": "4.9",
        "profile": "du"
    },
    "cpuset": "0-3,28-31",
    "rootdevicehints_devicename": "/dev/sda",
    "ssh_public_key": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDA64h5TYyK84L/e0/a5wtGmB1TRlnQNFgkZZpC7qSyWloD2bujfbbYb5oN8s8YHUHA2+QU4pI4V9fJk/K0X5f8Pte+Oq+EVJZznUULxi496VtnXnqFGG7zPSADMXMI4SFXMnHz2r4Akvwrg6qzoxzMctHGb7VDF+fWOuuarcvQBXqiGSu/GTHkrELMp1FBKrrb2dMYXvSua/dDMbV3DQthGfWF4EJQEPAyb+5iAdo0sWk8l4betMn5LqClnvDOEIUyrEEOWusBpKqXhQP6Wz3MvjCFITLXRKjEd+ti0l0SIKh47nGxktYyAqB8QkVx1ijlPmL0n7I9XSkQhHIAA4SBKOH7yzxgqqpPW7DSZF+YV/c/yFbTjSoSK/xd4x5Miapvnkj9Ewh6P6AUkVvDU5Ipf5SyPvvScI0IseiBhRqC8w8nAYx88oCLDCbMf8FFw962IOKwGvrJCUnbsAwjbUhtFMSUNXQ+5QdIxxUkR2KIoES+V6Hd6iYnkCovt04soV0= root@vranmgmt-dhcp-server6.rtp.raleigh.ibm.com"
}
```
Create Management Site and map to dns server, git server and provisioner_server

	Do not add roles to management config context data, SNO config context data

	For Site, no need of region

	Do not add region in image registry config
	For Automation context, add descriptor as ocp-automation 1.0

	Download latest code for ocp-cluster-automation-develop.zip sent by Raghav
	Insert tnco namespace for cp4na details under the management cluster config  context
```yaml
	      "cp4na": {
            "api_key": "hqxfCZPxoCeYAiL9V7tvZTVtAulhBK4Tt551QPcy",
            "ocp_password": "EXT.Dem0s",
            "ocp_server": "api.tnc-env13-rhocp.nca.ihost.com:6443",
            "ocp_user": "tncodevteam",
            "tnco_namespace": "lifecycle-manager",
            "username": "admin"
        },
```
Deployment location not found” - error:

	Place devices other than site into Management.

	Make sure sudo global config is done for github creds and commits
```sh
oc apply -f assisted-deployment-pull-secret.yaml for the open-cluster-management ddinamespace
oc get secret platform-auth-idp-credentials -n ibm-common-services -o yaml
```
To enter vault
```sh
​​oc get secret cp4na-o-vault-keys -o jsonpath='{.data.alm_token}' | base64 -d
```
For extending timeout:
In daytona
```
alm.daytona.resource-manager.default-timeout-duration": "10000s,
```
In brent
```yaml
{
  "alm.brent.asynchronousRequests.topic": "tnco-rm-transition-request",
  "alm.brent.asynchronousResponses.topic": "tnco-rm-transition-response",
  "alm.brent.driverContextRepository.ttl": "3",
  "alm.brent.topics.descriptorChangeTopic": "tnco-descriptor-change",
  "alm.brent.topics.lifecycleResponsesTopic": "lm_vnfc_lifecycle_execution_events",
  "alm.brent.transitionsRepository.ttl": "3",
  "alm.brent.vault.host": "cp4na-o-vault.lifecycle-manager",
  "spring.datasource.url": "jdbc:postgresql://cp4na-o-postgresql-rw.lifecycle-manager.svc.cluster.local:5432/app?currentSchema=cp4na",
  "": ""
}
```
Delete daytona pod

	Make sure in brent the above data is same incase any change restart brent pod

	Config context
```yaml
{
    "management_cluster_config": {
        "argocd": {
            "namespace": "openshift-gitops",
            "route_name": "openshift-gitops-server",
            "secret_name": "openshift-gitops-cluster",
            "url": "openshift-gitops-server-openshift-gitops.apps.tnc-env13-rhocp.nca.ihost.com"
        },
        "cp4na": {
            "api_key": "hqxfCZPxoCeYAiL9V7tvZTVtAulhBK4Tt551QPcy",
            "ocp_password": "EXT.Dem0s",
            "ocp_server": "api.tnc-env13-rhocp.nca.ihost.com:6443",
            "ocp_user": "tncodevteam",
            "tnco_namespace": "lifecycle-manager",
            "username": "admin"
        },
        "max_timeout": "90",
        "ntp_server": "10.0.10.72",
        "rhacm": {
            "password": "EXT.Dem0s",
            "server": "api.tnc-env13-rhocp.nca.ihost.com:6443",
            "user": "tncodevteam"
        }
    }
}
```

	FOR DEBUG

	To do soft teardown:-
```yaml
oc exec cp4na-o-galileo-0 -- curl -XDELETE https://localhost:8283/api/topology/assemblies/8e9b7c5c-5b77-441b-9980-3b7c886c2f55 --insecure -n ibmcp4na
```
Argocd login:
```sh
sudo argocd login
cd sbin
```
Change secret from argocd-secret to openshift-gitops-cluster:-
```yaml
{
    "management_cluster_config": {
        "argocd": {
            "namespace": "openshift-gitops",
            "route_name": "openshift-gitops-server",
            "secret_name": "openshift-gitops-cluster",
            "url": "openshift-gitops-server-openshift-gitops.apps.tnc-env13-rhocp.nca.ihost.com"
        },
        "cp4na": {
            "api_key": "hqxfCZPxoCeYAiL9V7tvZTVtAulhBK4Tt551QPcy",
            "ocp_password": "EXT.Dem0s",
            "ocp_server": "api.tnc-env13-rhocp.nca.ihost.com:6443",
            "ocp_user": "tncodevteam",
            "tnco_namespace": "lifecycle-manager",
            "username": "admin"
        },
        "max_timeout": "90",
        "ntp_server": "10.0.10.72",
        "rhacm": {
            "password": "EXT.Dem0s",
            "server": "api.tnc-env13-rhocp.nca.ihost.com:6443",
            "user": "tncodevteam"
        }
    }
}
```

	If API endpoint is not being reached:
	Remove last four lines in dnsmasq.conf and restart dnsmasq

	Also connect argocd app to github repo [github repo link and token] under repositories section of ArgoCD

	Sync the site-management app with force ticked as prune option
	Painpoints

	* Create assisted-deployment-pull-secret in open-cluster-management namespace
	* Add ZTP github repo in argocd
	*Add host entries in /etc/hosts


https://access.redhat.com/solutions/6824171 [bmh stuck in preparing state]

