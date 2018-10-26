# Prepare Phase

**[Setup Ansible Script Repository](./how-to-use.md)** - *Use 3.9 branch*

## Check List
- Validate OpenShift Container Platform storage migration
  ```
   $ oc adm migrate storage --include=* --loglevel=2 --confirm 
  ```

- Remove `openshift_schedulable` for each master from ansible inventory file


- Update version related parameter in inventory file 

```
  openshift_release="3.9"
  openshift_version="3.9"
          ....
          ....
  ```

- [Disable SWAP](https://docs.openshift.com/container-platform/3.9/admin_guide/overcommit.html#disabling-swap-memory)
  ```
  ansible -i /etc/ansible/hosts all -m shell -a "swapoff -a"
  ```

- By default, only openshift service will be restarted. If you want to restart system(node), specify below in inventory file
  ```
  openshift_rolling_restart_mode=system
  ``` 

- Update ansible inventory file if you did some manual configuration.(ex, admissionConfig)

- After upgrade, reboot all hosts

## Export Environment Variables
```
export API_SERVER=https://openshift.example.com:8443
```

## Update variables
Do not change the default values from “./vars/defaults.yml” 
If you want to override default variable, please use “./vars/override.yml” file. One thing you have to check is the latest version of image.
```
$ vi vars/override.yml

# Metrics
#openshift_metrics_image_version=<tag>
#openshift_metrics_hawkular_hostname=<fqdn>
#openshift_metrics_cassandra_storage_type=(emptydir|pv|dynamic)
#metrics_image_version: v3.9.40
#metrics_hawkular_hostname:
#metrics_cassandra_storage_type: emptydir

# EFK
#openshift_logging_image_version == efk_image_version
#efk_image_version: v3.9.40

# Node Upgrade
#openshift_upgrade_master_nodes_serial: "50%"
#openshift_upgrade_infra_nodes_serial: "1"
#openshift_upgrade_app_nodes_serial: "20%"
#openshift_upgrade_infra_nodes_label: "region=infra"
#openshift_upgrade_app_nodes_label: "region=app"

## Node Upgrade Common variables
#openshift_upgrade_nodes_max_fail_percentager: 20
#openshift_upgrade_nodes_drain_timeout: 600

```


## Upgrade Phase 

*Pre-requisites: Ansible Playbook: prepare_for_upgrade.yml*
- Refresh subscription (for all nodes)
- Change repositories (3.9)
- Reinstall ansible-openshift-utils on Ansible Controller

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/prepare_for_upgrade.yml 
```

## Validate Prerequisites 
```
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml
```

### Upgrade to the latest version of OCP 3.9 Control Plane
*Commands*
```

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,control_plane --skip-tags efk,metrics -e @vars/default.yml -e @vars/override.yml -vvvv
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

### Upgrade to the latest version of OCP 3.9 Infra Nodes 
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_infra_nodes_serial: "1" 
openshift_upgrade_infra_nodes_label: "region=infra"


$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,infra --skip-tags efk,metrics -e @vars/default.yml -e @vars/override.yml 
```

### Upgrade to the latest version of OCP 3.9 App Nodes
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_app_nodes_serial: "20%" 
openshift_upgrade_app_nodes_label: "region=app"


$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,app --skip-tags efk,metrics -e @vars/default.yml -e @vars/override.yml 

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

logging-curator-v3.7.61-2
logging-elasticsearch-v3.7.61-2
...
logging-fluentd-v3.7.61-2
logging-kibana-v3.7.61-2
logging-auth-proxy-v3.7.61-2

```

*Commands*
```
$ cat vars/default.yml
...
efk_image_version: 3.7.61

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,efk --skip-tags metrics -e @vars/default.yml -e @vars/override.yml 
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
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade.yml --tag always,metrics --skip-tags efk -e @vars/default.yml -e @vars/override.yml 
```

#### New container image version check
```
$ oc get po -n openshift-infra -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec {{$pod.metadata.name}} -- find /root/buildinfo -name Dockerfile-openshift* | grep -o metrics.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

metrics-cassandra-v3.9.40-11
metrics-hawkular-metrics-v3.9.40-11
metrics-heapster-v3.9.40-11

```

#### [Verify latest version of OCP](./verify-ocp-health.md)
