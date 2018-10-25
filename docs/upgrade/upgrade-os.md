Upgrade OS
----------


### [Setup Ansible Script Repository](./how-to-use.md)

### Update default variables
```
cat ./vars/default.yml
…
openshift_upgrade_master_nodes_serial: "50%"
openshift_upgrade_infra_nodes_serial: "1" 
openshift_upgrade_app_nodes_serial: "20%" 
openshift_upgrade_infra_nodes_label: "region=infra"
openshift_upgrade_app_nodes_label: "region=app"
```

### Execute Ansible Script
```
# Masters
ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade-os.yml  --tag always,master  -e @vars/default.yml

# Infra nodes
ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade-os.yml  --tag always,infra  -e @vars/default.yml

# App nodes
ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade-os.yml  --tag always,app  -e @vars/default.yml
```

### Result
- Check kernel version

Ex) RHEL 7.3 to 7.5

```
# 7.3 RHEL kernel version

$ uname -r
3.10.0-514.26.2.el7.x86_64

# 7.5 RHEL kernel version
3.10.0-862.11.6.el7.x86_64

#Check all nodes
ansible all -i /etc/ansible/hosts -m command -a "uname -r"

```

