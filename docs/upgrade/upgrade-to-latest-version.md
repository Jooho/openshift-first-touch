# Prepare Phase

**[Setup Ansible Script Repository](./how-to-use.md)** - *Use 3.7 branch*

## Check List
- ETCD v3?
- openshift_deployment_type=openshift-enterprise
- openshift_rolling_restart_mode=system (services -> do not reboot node)
- Comment out `openshift_pkg_version`
- Validate OpenShift Container Platform storage migration 
  ```
  $ oc adm migrate storage --include=* --loglevel=2 --confirm --config /etc/origin/master/admin.kubeconfig
  ```


## Export Environment Variables
```
export API_SERVER=https://openshift.example.com:8443
```

## Update variables
Do not change the default values from “./vars/defaults.yml” 
If you want to override default variable, please use “./vars/override.yml” file. One thing you have to check is the latest version of image.
```
$ cd openshift-first-touch/ansible
$ vi vars/override.yml

## Upgrade - Metrics
#openshift_metrics_image_version=<tag>
#openshift_metrics_hawkular_hostname=<fqdn>
#openshift_metrics_cassandra_storage_type=(emptydir|pv|dynamic)
###############################################################
#metrics_image_version: v3.7.61
#metrics_hawkular_hostname:
#metrics_cassandra_storage_type: emptydir

## Upgrade - EFK
#openshift_logging_image_version == efk_image_version
#efk_image_version: v3.7.61

## Upgrade - Node
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

### Execute upgrade checking playbook
```
ansible-playbook -i /etc/ansible/hosts /usr/share/ansible/openshift-ansible/playbooks/byo/openshift-cluster/upgrades/v3_7/upgrade.yml --tags pre_upgrade

```

### Simple Inplace upgrade 
*Pre-requisites: Ansible Playbook: prepare_for_upgrade.yml*
- Refresh subscription (for all nodes)
- Change repositories (default 3.7)
- Reinstall ansible-openshift-utils on Ansible Controller

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/prepare_for_upgrade.yml 
```

### Upgrade to the latest version of OCP 3.7 Control Plane
*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,control_plane --skip-tags efk,metrics -e @vars/default.yml -vvvv
```

### Check Control Plane
```
$ curl -k $API_SERVER/healthz
ok
$ curl -k $API_SERVER/version
{
  "major": "1",
  "minor": "7",
  "gitVersion": "v1.7.6+a08f5eeb62",
  "gitCommit": "c84beff",
  "gitTreeState": "clean",
  "buildDate": "2018-07-30T13:34:57Z",
  "goVersion": "go1.8.3",
  "compiler": "gc",
  "platform": "linux/amd64"
}

$ curl -k $API_SERVER/version/openshift
{
  "major": "3",
  "minor": "7",
  "gitCommit": "52f9778",
  "gitVersion": "v3.7.61",
  "buildDate": "2018-07-30T13:34:57Z"
}
```

### Upgrade to the latest version of OCP 3.7 Infra Nodes 
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_infra_nodes_serial: "1" 
openshift_upgrade_infra_nodes_label: "region=infra"


$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade-to-latest-version.yml --tag always,infra --skip-tags efk,metrics -e @vars/default.yml 
```

### Upgrade to the latest version of OCP 3.7 App Nodes
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_app_nodes_serial: "20%" 
openshift_upgrade_app_nodes_label: "region=app"

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade/upgrade-to-latest-version.yml --tag always,app --skip-tags efk,metrics -e @vars/default.yml 
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

logging-elasticsearch-v3.7.52-1
….
logging-fluentd-v3.7.52-1
logging-kibana-v3.7.52-2
logging-auth-proxy-v3.7.52-1
```

*Commands*
```
$ cat vars/default.yml
...
efk_image_version: 3.7.61

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,efk --skip-tags metrics -e @vars/default.yml 
```

#### New container image version check
```
$ oc get po -n logging -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec -c {{$container.name}} {{$pod.metadata.name}} -n logging -- find /root/buildinfo -name Dockerfile-openshift* | grep -o logging.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

logging-curator-v3.7.61-2
logging-elasticsearch-v3.7.61-2
...
logging-fluentd-v3.7.61-2
logging-kibana-v3.7.61-2
logging-auth-proxy-v3.7.61-2
```

### Upgrade the Cluster Metrics

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,metrics --skip-tags efk -e @vars/default.yml 
```

#### New container image version check
```
$ oc get po -n openshift-infra -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec {{$pod.metadata.name}} -- find /root/buildinfo -name Dockerfile-openshift* | grep -o metrics.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

metrics-cassandra-v3.7.61-11
metrics-hawkular-metrics-v3.7.61-11
metrics-heapster-v3.7.61-11
```

### Upgrade Service Catalog

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,catalog --skip-tags efk -e @vars/default.yml 
```


[Verify latest version of OCP](./verify-ocp-health.md)
