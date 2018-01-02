# Red Hat OpenShift on HPE Synergy Playbooks

This is a set of ansible playbooks to provision HPE Synergy modules in preparation for deploying a Red Hat OpenShift cluster. These playbooks use HPE OneView and the associated ansible modules to automate the provisioning. They are meant to  be run from an Ansible Tower server, however can be run normally with `ansible-playbook` with some slight modifications.

The setup assumes using an Ansible Tower server however they could also refer to a 'control' node if not using Tower.

**NOTE** The playbooks provided in the GitHub repository are not supported by Red Hat. They merely provide a mechanism that can be used to build out an OpenShift Container Platform environment using HPE OneView.

# Setup

The playbooks in this repo are to support a [Red Hat Reference Architcture](https://access.redhat.com/documentation/en/reference-architectures) in which the full explanation of usage of these playbooks is provided.

The Red Hat reference architecture is still a work in progress, but the HPE branded version can be found [on HPE's website](https://h20195.www2.hpe.com/V2/GetDocument.aspx?docname=a00038916enw).

These playbooks assume that an Ansible Tower and Red Hat Satellite server has already been deployed in the environment, and use an existing HPE OneView/Synergy environment to provision servers. There are a number of artifacts (golden images, build plans, scripts, etc) that will need to be added to your Synergy environment. Those artifacts are available [here](https://github.com/HewlettPackard/image-streamer-reference-architectures/tree/master/RC-RHEL-OpenShift).

Refer to the reference architecture and artifact repo on how to set those up.

## Installation

On the Tower server, a few additional packages are required:

~~~
yum install python2-jmespath python-dns httpd-tools java-1.8.0-openjdk-headless ansible-tower-cli
~~~

The HPE OneView SDK is also required, which can be installed via `pip`:

~~~
curl -o get-pip.py https://bootstrap.pypa.io/get-pip.py
python get-pip.py
pip install hpOneView dnspython
~~~

The HPE OneView modules do not currently ship with ansible, and must be installed separately:

~~~
git clone https://github.com/HewlettPackard/oneview-ansible.git /tmp/oneview-ansible
mkdir -p /usr/share/ansible/oneview-ansible
rsync -avh --progress /tmp/oneview-ansible/library /usr/share/ansible/oneview-ansible
~~~

## Configuration

A sample inventory and group variable files are included in this repo under `./inventory`. In addition, a number of account credentials for the integrated services (OneView, Satellite, etc) are required. If a local git repository is available, this repo should be pulled in and modified there. Otherwise, clone the repo to the Tower server and follow the below steps.

For the inventory and variables:

~~~
mkdir /etc/ansible/ocp-on-synergy
rsync -avh --progress /path/to/git/repo/inventory/* /etc/ansible/ocp-on-synergy  # sample inventory files
chmod -R awx:awx /etc/ansible/ocp-on-synergy
~~~

Then modify `/etc/ansible/ocp-on-synergy/hosts` and ``/etc/ansible/ocp-on-synergy/group_vars/*`` to match the target environment. To keep things simple and allow the bare-metal provisioning and OpenShift Ansible playbooks to share an inventory, the required inventory file and variables for the openshift-ansible playbooks is extended. Full details concerning the OpenShift specific variables is covered in the [Advanced Install](https://docs.openshift.org/latest/install_config/install/advanced_install.html#configuring-ansible) guide for Red Hat OpenShift.

The added variables define how the nodes should be provisioned, such as network information for the nodes, server profiles and OS deployment plans in OneView and credentials to access required services.

This is sample environment, where direct control over the placement of the initial nodes is desired for redundancy purposes:

~~~
[tower]
ansible.example.com

[OSEv3:children]
nodes
masters
etcd
lb
glusterfs
nfs
new_nodes

[nodes]
ocp-master1.example.com  openshift_schedulable=False ansible_host=192.168.1.112 enclosure_name='FRM1-TOP: CN1234567A' enclosure_bay=1
ocp-master2.example.com  openshift_schedulable=False ansible_host=192.168.1.113 enclosure_name='FRM2-MID: CN1234567B' enclosure_bay=1
ocp-master3.example.com  openshift_schedulable=False ansible_host=192.168.1.114 enclosure_name='FRM3-BTM: CN1234567C' enclosure_bay=1
ocp-infra1.example.com  openshift_node_labels="{'region': 'infra'}" ansible_host=192.168.1.115 enclosure_name='FRM1-TOP: CN1234567A' enclosure_bay=2
ocp-infra2.example.com  openshift_node_labels="{'region': 'infra'}" ansible_host=192.168.1.116 enclosure_name='FRM2-MID: CN1234567B' enclosure_bay=2
ocp-infra3.example.com  openshift_node_labels="{'region': 'infra'}" ansible_host=192.168.1.117 enclosure_name='FRM3-BTM: CN1234567C' enclosure_bay=2
ocp-cns1.example.com  ansible_host=192.168.1.118 enclosure_name='FRM1-TOP: CN1234567A' enclosure_bay=7
ocp-cns2.example.com  ansible_host=192.168.1.119 enclosure_name='FRM2-MID: CN1234567B' enclosure_bay=7
ocp-cns3.example.com  ansible_host=192.168.1.120 enclosure_name='FRM3-BTM: CN1234567C' enclosure_bay=7
ocp-node1.example.com  ansible_host=192.168.1.121 enclosure_name='FRM1-TOP: CN1234567A' enclosure_bay=8
ocp-node2.example.com  ansible_host=192.168.1.122 enclosure_name='FRM2-MID: CN1234567B' enclosure_bay=8
ocp-node3.example.com  ansible_host=192.168.1.123 enclosure_name='FRM3-BTM: CN1234567C' enclosure_bay=8

[masters]
ocp-master1.example.com
ocp-master2.example.com
ocp-master3.example.com

[infra]
ocp-infra1.example.com
ocp-infra2.example.com
ocp-infra3.example.com

[etcd]
ocp-master1.example.com
ocp-master2.example.com
ocp-master3.example.com

[lb]
ocp-lbnfs.example.com ansible_host=192.168.1.111 enclosure_name='FRM3-BTM: CN1234567C' enclosure_bay=11
[nfs]
ocp-lbnfs.example.com

[glusterfs]
ocp-cns1.example.com glusterfs_ip=192.168.1.118  glusterfs_devices='["/dev/sdb"]'
ocp-cns2.example.com glusterfs_ip=192.168.1.119  glusterfs_devices='["/dev/sdb"]'
ocp-cns3.example.com glusterfs_ip=192.168.1.120  glusterfs_devices='["/dev/sdb"]'

[new_nodes]
~~~

NOTE: The `ansible_host` variable will become the **static IP** for the provisioned node.

In `/etc/ansible/ocp-on-synergy/group_vars/all`, tailor the file to match the profiles, OS deployment plans and networking information used in OneView:

~~~
---
master_deployment_plan: "HPE-RHEL7.4-OpenShiftMaster"
worker_deployment_plan: "HPE-RHEL7.4-OpenShiftWorker"
cns_deployment_plan: "HPE-RHEL7.4-OpenShiftCNS"
infra_deployment_plan: "HPE-RHEL7.4-OCP-Infra"
lbnfs_deployment_plan: "HPE-RHEL7.4-OpenShiftLBNFS"
master_server_profile_template: "OCP-Master"
worker_server_profile_template: "OCP-Worker"
cns_server_profile_template: "OCP-CNS-Node"
lbnfs_server_profile_template: "OCP-LB-NFS"
infra_server_profile_template: "OCP-Infra"
deployment_plan: "{{ worker_deployment_plan }}"
server_profile_template: "{{ worker_server_profile_template }}"
## ocp
openshift_master_default_subdomain: paas.example.com
openshift_master_cluster_hostname: ocp-cluster.example.com
openshift_master_cluster_public_hostname: ocp-cluster.example.com
## oneview vars
oneview_auth:
  ip: "192.168.1.10"
  username: "Administrator"
  api_version: 300
## network settings
network:
  name: "MGMT1"
  gateway: "192.168.1.1"
  netmask: "255.255.255.0"
  domain: "example.com"
  dns1: "192.168.1.106"
  dns2: "192.168.1.104"
dns_server: "{{ network.dns1 }}"
## satellite
satellite_user: "admin"
satellite_url: "https://192.168.1.106"
inventory_path: "/etc/ansible/ocp-on-synergy"
~~~

NOTE: Additional variables are set under `./roles/provisioning/vars/main.yaml` and should not have to be changed if the reference architecure is followed. If the target environment differs from the associated RA, then make changes to that file as appropriate.

With all the variables set, the inventory can now be imported to Tower. First create an empty inventory in the GUI e.g. *ocp-on-synergy* then run the following to import to tower:
~~~
tower-manage inventory_import --source /etc/ansible/ocp-on-synergy/ --inventory-name=ocp-on-synergy --overwrite --overwrite-vars
~~~

These playbooks use `ansible-vault` to encrypt sensitive login credentials. Modify passwords.yaml and vault it like so:

~~~
$ ansible-vault encrypt passwords.yml --output=roles/passwords/vars/passwords.yml
$ chmod 644 roles/passwords/vars/passwords.yml
~~~

# Tower Configuration

In the previous step, an inventory should have been created in Tower. To knit the playbooks together and execute via tower, some additional steps are required.

First, the two **projects** need to be defined. Under the PROJECTS tab in the GUI, create 2 projects. One will contain the provisioning playbooks included in this repo stored locally or on an internal git server, while the other will point to the openshift-ansible repository on [github](https://github.com/openshift/openshift-ansible).

A root ssh key and the password used to encrypt the credentials with ansible-vault in the previous step must be stored in Tower. Click the 'gear' icon in the Tower GUI to access the settings menu, and click credentials. Click on 'add' to create a new credential object. Create a credential of type 'machine' to store the SSH private key from the tower server, and another of type 'vault' for the vault password. The SSH key should be the same key defined in HPE OneView, injected by the Image Streamer build scripts. The names **root-ssh** and **ocp-vault** are suggestions and referred to in further configuration steps.

## Job and Workflow Templates

### Base Deployment

Between the two projects used, there are a number of playbooks that need to be executed. They in turn can be chained together in a workflow. Create the following job templates in Tower. The table lists values that need to be changed. The default value for the remaining options are fine.

| Name        | Description  | Inventory  | Project | Playbook | Credential | Forks
| ------------- |:-------------:| -----:| -----:| -----:| -----:| -----:|
| ocp-cleanup      | clean up previous deployment | ocp-on-synergy | ocp-on-synergy | playbooks/clean.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 20 |
| ocp-provisioning      | deploy bare-metal nodes via HPE OneView | ocp-on-synergy | ocp-on-synergy | playbooks/provisioning.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 20 |
| ocp-predeploy      | pre-deployment tasks for OCP | ocp-on-synergy | ocp-on-synergy | playbooks/predeploy.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 10 |
| ocp-openshift-install      | deploy openshift on synergy nodes | ocp-on-synergy | openshift-ansible | playbooks/byo/config.yml | MACHINE: root-ssh | 20 |
| ocp-postdeploy      | post-deployment tasks for OCP | ocp-on-synergy | ocp-on-synergy | playbooks/postdeploy.yaml | MACHINE: root-ssh | 10 |

NOTE: the `ocp-cleanup` job is handy if the environment will be rebuilt numerous times. It can be safely omitted if that is not the case.


They can now be combined in to a workflow. Under the Templates screen, click `+ADD -> Workflow Template`. Give it a name and a description, then open up the `Workflow Editor`. Use the editor to chain the playbooks together. Click stat and select the **ocp-cleanup** job from the list. Then hit the `+` and continue to add the remaining playbooks. The workflow order should be:

`ocp-cleanup` -> `ocp-provisioning` -> `ocp-predeploy` -> `ocp-openshift-install` -> `ocp-postdeploy`

### Scale Up and Down

This repo coupled with the openshift-ansible playbooks can also be used to scale up or down a cluster. Some additional job and workflow templates are required. Note that for the `ocp-provisioning-limited` playbook, a limit is applied so the job only runs against the new server.

| Name        | Description  | Inventory  | Project | Playbook | Credential | Forks | Limit |
| ------------- |:-------------:| -----:| -----:| -----:| -----:| -----:| -----:|
| ocp-add_to_oneview      | add new node to inventory and oneview | ocp-on-synergy | ocp-on-synergy | playbooks/add_to_oneview.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 20 | /blank/ |
| ocp-provisioning-limited      | deploy bare-metal node via HPE OneView | ocp-on-synergy | ocp-on-synergy | playbooks/provisioning.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 10 | new_nodes |
| ocp-openshift-scaleup      | deploy openshift on synergy node | ocp-on-synergy | openshift-ansible | playbooks/byo/openshift-node/scaleup.yml | MACHINE: root-ssh | 20 | /blank/ |
| ocp-fix_inventory      | remove new node from new_nodes group | ocp-on-synergy | ocp-on-synergy | playbooks/fix_inventory.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 10 | /blank/ |
| ocp-scaledown      | remove a node from ocp and synergy | ocp-on-synergy | ocp-on-synergy | playbooks/scaledown.yaml | MACHINE: root-ssh, VAULT: ocp-vault | 10 | /blank/ |

Create a new workflow for the scaleup, similar to the base deployment, with the following steps:

`ocp-add_to_oneview` -> `ocp-provisioning-limited` -> `ocp-openshift-scaleup` -> `ocp-fix_inventory`

This workflow expects to be supplied with a host name and IP for the new node. This variables can be supplied via a **survey** in the workflow. Click the `ADD SURVEY` button.  Configure the following two prompts:

| Prompt        | Description  | Answer Variable Name  | Answer Type | Required |
| ------------- |:-------------:| -----:| -----:| -----:|
| New Hostname       | enter a hostname for the new node  | new_node | String | Checked |
| New IP       | enter the IP address for the new node  | new_ip | String | Checked |


Similarly, the `scaledown` job also requires a survey. Configure that job to prompt for a **target_node**:

| Prompt        | Description  | Answer Variable Name  | Answer Type | Required |
| ------------- |:-------------:| -----:| -----:| -----:|
| Target Hostname       | enter a hostname to remove from the cluster  | target_node | String | Checked |


# Run it

With all that out of the way, the jobs can now be executed from Tower. Select the `ocp-on-synergy` workflow, hit the rocket ship button and go for a coffee. In a little while (1-2hrs), a fully deployed Red Hat OpenShift Container Platform cluster will be available.

The scaleup and scaledown jobs can be launched in a similar manner, as demand changes. When scaling up the environment, the user will be prompted for the new hostname and IP after hitting the rocket ship, and in about 15 minutes the new node will be in the cluster. Scaledown will prompt for the node to remove and should complete in around 5 minutes.
