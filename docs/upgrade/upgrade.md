# Prepare Phase

**[Setup Ansible Script Repository](./how-to-use.md)** - *Use 3.10 branch*


## Check List
- Validate OpenShift Container Platform storage migration
  ```
   $ oc adm migrate storage --include=* --loglevel=2 --confirm 
  ```

- Change Node Label to [Node Group](https://docs.openshift.com/container-platform/3.10/upgrading/automated_upgrades.html#upgrades-defining-node-group-and-host-mappings)
  ```
  openshift_node_groups=[{'name': 'node-config-master', 'labels': ['node-role.kubernetes.io/master=true']}, {'name': 'node-config-infra', 'labels': ['node-role.kubernetes.io/infra=true',]}, {'name': 'node-config-compute', 'labels': ['node-role.kubernetes.io/compute=true'], 'edits': [{ 'key': 'kubeletArguments.pods-per-core','value': ['20']}]}]
  ```

- `openshift_hostname` parameter is removed so remove it from inventory file. Instead, `openshift_kubelet_name_override` must be set with the value of `openshift_hostname` that you used in previous versions.


- Update version related parameter in inventory file 
 
  ```
  openshift_release="3.10"
  openshift_version="3.10"
          ...
          ...
  ```
- BackUp
  - Master (upgrade_prepare ansible will do it)
  ```
  /usr/lib/systemd/system/atomic-openshift-master-api.service
  /usr/lib/systemd/system/atomic-openshift-master-controllers.service
  /etc/sysconfig/atomic-openshift-master-api
  /etc/sysconfig/atomic-openshift-master-controllers
  /etc/origin/master/master-config.yaml
  /etc/origin/master/scheduler.json
  ```
  - Node (upgrade_prepare ansible will do it)
  ```
  /usr/lib/systemd/system/atomic-openshift-*.service
  /etc/origin/node/node-config.yaml
  ```
  - [ETCD](https://docs.openshift.com/container-platform/3.10/day_two_guide/environment_backup.html#etcd-backup_environment-backup) Config (upgrade_prepare ansible will do it)
  ```
  /etc/etcd/etcd.conf 
  
   ssh master-0
   mkdir -p /backup/etcd-config-$(date +%Y%m%d)/
   cp -R /etc/etcd/ /backup/etcd-config-$(date +%Y%m%d)/
  ```
  - **MANUAL TASK** [ETCD DB](https://docs.openshift.com/container-platform/3.10/day_two_guide/environment_backup.html#etcd-backup_environment-backup)

  
- By default, only openshift service will be restarted. If you want to restart system(node), specify below in inventory file
  ```
  openshift_rolling_restart_mode=system
  ``` 

- Update ansible inventory file if you did some manual configuration.(ex, admissionConfig)

- After upgrade, reboot all hosts


## Noticeable Change

- Node ConfigMaps
  node configurations are now [bootstrapped](https://docs.openshift.com/container-platform/3.10/architecture/infrastructure_components/kubernetes_infrastructure.html#node-bootstrapping) from the master. Do *NOT* change /etc/origin/node/node-config.yaml m
anually. It will be overrided by ConfigMap.

  ```
  Changes should not be made to a node host’s /etc/origin/node/node-config.yaml file. They will be overwritten by the configuration defined in the ConfigMap used by the node.
  ```
  - default node group (/usr/share/ansible/openshift-ansible/roles/openshift_facts/defaults/main.yml) 
## Export Environment Variables
```
export API_SERVER=https://openshift.example.com:8443
```

## Update variables
If you want to override default variable, please use “./vars/default.yml” file. One thing you have to check is the latest version of image.
```
$ vi vars/default.yml

## Upgrade - Metrics
#openshift_metrics_image_version=<tag> 
#openshift_metrics_hawkular_hostname=<fqdn> 
#openshift_metrics_cassandra_storage_type=(emptydir|pv|dynamic) 
###############################################################
metrics_image_version: v3.10
#metrics_hawkular_hostname:
#metrics_cassandra_storage_type: emptydir

## Upgrade - EFK
#openshift_logging_image_version == efk_image_version
efk_image_version: v3.10

## Upgrade - Node
openshift_upgrade_master_nodes_serial: "50%" 
openshift_upgrade_infra_nodes_serial: "1" 
openshift_upgrade_app_nodes_serial: "20%" 
openshift_upgrade_infra_nodes_label: "region=infra"
openshift_upgrade_app_nodes_label: "region=app"

## Node Upgrade Common variables
openshift_upgrade_nodes_max_fail_percentager: 20 
openshift_upgrade_nodes_drain_timeout: 600

```

## Practical Updates for Ansible hosts file
- Change node label to node group
```
openshift_node_groups=[{'name': 'node-config-master', 'labels': ['node-role.kubernetes.io/master=true', 'role=master','region=mgmt']}, {'name': 'node-config-infra', 'labels': ['node-role.kubernetes.io/infra=true','role=infra','region=infra','zone=default']}, {'name': 'node-config-compute', 'labels': ['node-role.kubernetes.io/compute=true', 'role=app','region=app', 'zone=default'], 'edits': [{ 'key': 'kubeletArguments.pods-per-core','value': ['20']}]}]

master.example.com openshift_node_group_name='node-config-master'
infra-node.example.com openshift_node_group_name='node-config-infra'
app-node.example.com openshift_node_group_name='node-config-compute'

```

- Change audit log path under /var/lib/origin
```
mkdir -p /var/lib/origin/log/openpaas-oscp-audit

vi /etc/ansible/hosts
...
openshift_master_audit_config={"enabled": true, "auditFilePath": "/var/lib/origin/log/openpaas-oscp-audit/openpaas-oscp-audit.log", "maximumFileRetentionDays": 14, "maximumFileSizeMegabytes": 500, "maximumRetainedFiles": 5}
```

- Metrics Storage Type
```
openshift_metrics_storage_kind=dynamic ==> openshift_metrics_cassandra_storage_type=dynamic
```


## Upgrade Phase 

* ETCD BACKUP is Manual JOB!!!! - DO IT Before upgrading*

*Pre-requisites: Ansible Playbook: prepare_for_upgrade.yml*
- Refresh subscription (for all nodes)
- Change repositories (3.10)
- Reinstall ansible-openshift-utils on Ansible Controller

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/prepare_for_upgrade.yml 
```

## Validate Prerequisites 
```
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
```

## Create ConfigMap for Node Configuration.
```
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/openshift-master/openshift_node_group.yml
```
## Verify the ConfigMap is the same as node-config.yaml
```
oc get configmaps -n openshift-node
```

## Updating Policy Definitions (If you have custom policy)
*You must manually update the roles that contain this setting to include any new or required permissions after upgrading.*

```
oc annotate clusterrole.rbac <role_name> --overwrite rbac.authorization.kubernetes.io/autoupdate=false

oc adm create-bootstrap-policy-file --filename=policy.json

# Update podlicy.json file
oc auth reconcile -f policy.json

oc adm policy reconcile-sccs --additive-only=true --confirm
```

### Upgrade to the latest version of OCP 3.10 Control Plane
*Commands*
```

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,control_plane --skip-tags efk,metrics -e @vars/default.yml -vvvv
```

#### Check Control Plane
```
$ curl -k $API_SERVER/healthz
ok
$ curl -k $API_SERVER/version
{
  "major": "1",
  "minor": "9",
  "gitVersion": "v1.9.1+a0ce1bc657",
  "gitCommit": "a0ce1bc",
  "gitTreeState": "clean",
  "buildDate": "2018-08-20T21:57:04Z",
  "goVersion": "go1.9.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}


$ curl -k $API_SERVER/version/openshift
{
  "major": "3",
  "minor": "9+",
  "gitCommit": "67432b0",
  "gitVersion": "v3.9.41",
  "buildDate": "2018-08-20T21:57:04Z"
}

```

### Upgrade to the latest version of OCP 3.10 Infra Nodes 
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_infra_nodes_serial: "1" 
openshift_upgrade_infra_nodes_label: "region=infra"


$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,infra --skip-tags efk,metrics -e @vars/default.yml -vvvv
```

### Upgrade to the latest version of OCP 3.10 App Nodes
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_app_nodes_serial: "20%" 
openshift_upgrade_app_nodes_label: "region=app"


$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,app --skip-tags efk,metrics -e @vars/default.yml -vvvv 

```


### Upgrade the EFK Logging stack

*Check List:*
- Fluentd DC/DS has following config, then change “IfNotPresent” to “Always”
```
image: <image_name>:<vX.Y>
imagePullPolicy: IfNotPresent
```

#### Existing container image version check
```
$ oc get po -n logging -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec -c {{$container.name}} {{$pod.metadata.name}} -n logging -- find /root/buildinfo -name Dockerfile-openshift* | grep -o logging.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

logging-curator-v3.9.40-2
logging-elasticsearch-v3.9.40-2
logging-fluentd-v3.9.40-2
…..
logging-kibana-v3.9.40-2
logging-auth-proxy-v3.9.40-2

```

*Commands*
```
$ cat vars/default.yml
...
efk_image_version: 3.9.10

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,efk --skip-tags metrics -e @vars/default.yml -vvvv
```

#### New container image version check
```
$ oc get po -n logging -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec -c {{$container.name}} {{$pod.metadata.name}} -n logging -- find /root/buildinfo -name Dockerfile-openshift* | grep -o logging.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

logging-curator-v3.9.40-2
logging-elasticsearch-v3.9.40-2
logging-fluentd-v3.9.40-2
…..
logging-kibana-v3.9.40-2
logging-auth-proxy-v3.9.40-2

```

### Upgrade the Cluster Metrics

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,metrics --skip-tags efk -e @vars/default.yml -vvvv
```

#### New container image version check
```
$ oc get po -n openshift-infra -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec {{$pod.metadata.name}} -- find /root/buildinfo -name Dockerfile-openshift* | grep -o metrics.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

metrics-cassandra-v3.9.40-11
metrics-hawkular-metrics-v3.9.40-11
metrics-heapster-v3.9.40-11

```

#### [Verify latest version of OCP](./verify-ocp-health.md)
