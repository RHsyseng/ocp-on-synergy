# Red Hat OpenShift on HPE Synergy Playbooks

This is a set of ansible playbooks to provision HPE Synergy modules in preparation for deploying a Red Hat OpenShift cluster. These playbooks use HPE OneView and the associated ansible modules to automate the provisioning.

**NOTE** The playbooks provided in the GitHub repository are not supported by Red Hat. They merely provide a mechanism that can be used to build out an OpenShift Container Platform environment using HPE OneView.

# Setup

The playbooks in this repo are to support a [Red Hat Reference Architcture](https://access.redhat.com/documentation/en/reference-architectures) in which the full explanation of the logic and usage of these playbooks is provided.

In brief:

Prepare environment:

~~~
yum install python2-jmespath python-pip httpd-tools java-1.8.0-openjdk-headless ansible-tower-cli
git clone https://github.com/HewlettPackard/oneview-ansible.git /tmp/oneview-ansible
mkdir -p /usr/share/ansible/oneview-ansible
rsync -avh --progress /tmp/oneview-ansible/library /usr/share/ansible/oneview-ansible
pip install hpOneView dnspython
~~~

Modify _inventory/group_vars/*_ to suit the target environment. Create an inventory file with the desired hostnames and designated IPs to use for the newly created nodes:

~~~
[OSEv3:children]
nodes
masters
etcd
lb
glusterfs
nfs

[nodes]
ocp-master1.cloud.example.com  openshift_schedulable=False ansible_host=192.168.173.52 enclosure_name=CN123000CK enclosure_bay=7
ocp-master2.cloud.example.com  openshift_schedulable=False ansible_host=192.168.173.53 enclosure_name=CN123000CK enclosure_bay=9
ocp-master3.cloud.example.com  openshift_schedulable=False ansible_host=192.168.173.54 enclosure_name=CN123000CK enclosure_bay=10
ocp-node1.cloud.example.com  openshift_node_labels="{'region': 'infra'}" ansible_host=192.168.173.49
ocp-node2.cloud.example.com  openshift_node_labels="{'region': 'infra'}" ansible_host=192.168.173.50
ocp-node3.cloud.example.com  openshift_node_labels="{'region': 'infra'}" ansible_host=192.168.173.51

[masters]
ocp-master1.cloud.example.com
ocp-master2.cloud.example.com
ocp-master3.cloud.example.com


[etcd]
ocp-master1.cloud.example.com
ocp-master2.cloud.example.com
ocp-master3.cloud.example.com

[lb]
lb.cloud.example.com
[nfs]
lb.cloud.example.com

[glusterfs]
ocp-node1.cloud.example.com glusterfs_ip=192.168.173.49  glusterfs_devices='["/dev/sdb"]'
ocp-node2.cloud.example.com glusterfs_ip=192.168.173.50  glusterfs_devices='["/dev/sdb"]'
ocp-node3.cloud.example.com glusterfs_ip=192.168.173.51  glusterfs_devices='["/dev/sdb"]'
~~~

These playbooks use ansible-vault to encrypt sensitive login credentials. Modify passwords.yaml and vault it like so:

~~~
$ ansible-vault encrypt passwords.yml --output=roles/passwords/vars/passwords.yml
$ chmod 644 roles/passwords/vars/passwords.yml
~~~

# Run it

~~~
$ ansible-playbook -i hosts playbooks/provisioning.yaml --ask-vault-pass
$ ansible-playbook -i hosts playbooks/predeploy.yaml --ask-vault-pass
~~~

These playbooks can also be used by Ansible Tower:

~~~
$ tower-manage inventory_import --source inventory/ --inventory-name=ocp-on-synergy --overwrite --overwrite-vars
~~~
pip install dnspython

tower-manage inventory_import --source /etc/ansible/hpe/ --inventory-name=ocp-on-synergy --overwrite --overwrite-vars
