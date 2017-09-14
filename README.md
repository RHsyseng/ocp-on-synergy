# Red Hat OpenShift on HPE Synergy Modules

This is a set of ansible playbooks to provision HPE Synergy modules in preparation for deploying a Red Hat OpenShift cluster. These playbooks use HPE OneView and the associated modules to automate the provisioning.

**NOTE** The playbooks provided in the GitHub repository are not supported by Red Hat. They merely provide a mechanism that can be used to build out an OpenShift Container Platform environment using HPE OneView.

# Setup

The playbooks in this repo are to support a [Red Hat Reference Architcture](https://access.redhat.com/documentation/en/reference-architectures) in which the full explanation of the logic and usage of these playbooks is provided.

In brief:

Prepare environment:

~~~
git clone https://github.com/HewlettPackard/oneview-ansible.git /tmp/oneview-ansible
mkdir -p /usr/share/ansible/oneview-ansible
rsync -avh --progress /tmp/oneview-ansible/library /usr/share/ansible/oneview-ansible
pip install hpOneView
~~~

Modify group_vars/* to suit the target environment. Create an inventory file with the desired hostnames and IPs that will be statically provisioned by Satellite:

~~~
## required for provisoning
ocp-master1.hpecloud.test ansible_host=192.168.2.173 ilo_ip=192.168.2.136
ocp-master2.hpecloud.test ansible_host=192.168.2.174 ilo_ip=192.168.2.137
ocp-master3.hpecloud.test ansible_host=192.168.2.175 ilo_ip=192.168.2.138
ocp-infra0.hpecloud.test ansible_host=192.168.2.172 ilo_ip=192.168.2.135
ocp-infra1.hpecloud.test ansible_host=192.168.2.170 ilo_ip=192.168.2.133
ocp-infra2.hpecloud.test ansible_host=192.168.2.171 ilo_ip=192.168.2.134
ocp-cns1.hpecloud.test ansible_host=192.168.2.176 ilo_ip=192.168.2.140
ocp-cns2.hpecloud.test ansible_host=192.168.2.177 ilo_ip=192.168.2.141
ocp-cns3.hpecloud.test ansible_host=192.168.2.178 ilo_ip=192.168.2.142
[cns]
ocp-cns1.hpecloud.test
ocp-cns2.hpecloud.test
ocp-cns3.hpecloud.test
~~~

These playbooks use ansible-vault to encrypt sensitive login credentials. Modify passwords.yaml and vault it like so:

~~~
$ ansible-vault encrypt passwords.yml --output=roles/passwords/vars/passwords.yml
~~~

# Run it

~~~
$ ansible-playbook -i hosts playbooks/provisioning.yaml --ask-vault-pass
~~~

yum install python2-jmespath.noarch
