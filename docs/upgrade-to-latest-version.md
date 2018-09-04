# Prepare Phase

<<<<<<< HEAD
**[Setup Ansible Script Repository](./how-to-use.md)** - *Use 3.7 branch*
=======
**[Setup Ansible Script Repository](./how-to-use.md)** - *Use 3.9 branch*
>>>>>>> 3.9

## Update variables
Do not change the default values from “./vars/defaults.yml” 
If you want to override default variable, please use “./vars/override.yml” file. One thing you have to check is the latest version of image.
```
$ vi vars/override.yml

<<<<<<< HEAD
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
=======
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
>>>>>>> 3.9
#openshift_upgrade_master_nodes_serial: "50%"
#openshift_upgrade_infra_nodes_serial: "1"
#openshift_upgrade_app_nodes_serial: "20%"
#openshift_upgrade_infra_nodes_label: "region=infra"
#openshift_upgrade_app_nodes_label: "region=app"

## Node Upgrade Common variables
#openshift_upgrade_nodes_max_fail_percentager: 20
#openshift_upgrade_nodes_drain_timeout: 600
<<<<<<< HEAD
=======

>>>>>>> 3.9
```


## Upgrade Phase 
<<<<<<< HEAD
### Simple Inplace upgrade 
*Pre-requisites: Ansible Playbook: prepare_for_upgrade.yml*
- Refresh subscription (for all nodes)
- Change repositories (default 3.7)
=======

*Pre-requisites: Ansible Playbook: prepare_for_upgrade.yml*
- Refresh subscription (for all nodes)
- Change repositories (3.9)
>>>>>>> 3.9
- Reinstall ansible-openshift-utils on Ansible Controller

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/prepare_for_upgrade.yml 
```

<<<<<<< HEAD
#### Upgrade to the latest version of OCP 3.7 Control Plane
=======
### Upgrade to the latest version of OCP 3.9 Control Plane
>>>>>>> 3.9
*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,control_plane --skip-tags efk,metrics -e @vars/default.yml -vvvv
```

#### Check Control Plane
```
$ curl $API_SERVER:8443/healthz
ok
$ curl $API_SERVER:8443/version
{
  "major": "1",
<<<<<<< HEAD
  "minor": "7",
  "gitVersion": "v1.7.6+a08f5eeb62",
  "gitCommit": "c84beff",
  "gitTreeState": "clean",
  "buildDate": "2018-07-30T13:34:57Z",
  "goVersion": "go1.8.3",
=======
  "minor": "9",
  "gitVersion": "v1.9.1+a0ce1bc657",
  "gitCommit": "a0ce1bc",
  "gitTreeState": "clean",
  "buildDate": "2018-08-20T21:57:04Z",
  "goVersion": "go1.9.4",
>>>>>>> 3.9
  "compiler": "gc",
  "platform": "linux/amd64"
}

<<<<<<< HEAD
$ curl $API_SERVER:8443/version/openshift
{
  "major": "3",
  "minor": "7",
  "gitCommit": "52f9778",
  "gitVersion": "v3.7.61",
  "buildDate": "2018-07-30T13:34:57Z"
}
```

#### Upgrade to the latest version of OCP 3.7 Infra Nodes 
=======

$ curl $API_SERVER:8443/version/openshift
{
  "major": "3",
  "minor": "9+",
  "gitCommit": "67432b0",
  "gitVersion": "v3.9.41",
  "buildDate": "2018-08-20T21:57:04Z"
}

```

### Upgrade to the latest version of OCP 3.9 Infra Nodes 
>>>>>>> 3.9
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_infra_nodes_serial: "1" 
openshift_upgrade_infra_nodes_label: "region=infra"


$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,infra --skip-tags efk,metrics -e @vars/default.yml 
```

<<<<<<< HEAD
#### Upgrade to the latest version of OCP 3.7 App Nodes
=======
### Upgrade to the latest version of OCP 3.9 App Nodes
>>>>>>> 3.9
*Commands*
```
$ cat vars/default.yml
...
openshift_upgrade_app_nodes_serial: "20%" 
openshift_upgrade_app_nodes_label: "region=app"

$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,app --skip-tags efk,metrics -e @vars/default.yml 
```


<<<<<<< HEAD
#### Upgrade the EFK Logging stack
=======
### Upgrade the EFK Logging stack
>>>>>>> 3.9

*Check List:*
- Fluentd DC/DS has following config, then change “IfNotPresent” to “Always”
```
image: <image_name>:<vX.Y>
imagePullPolicy: IfNotPresent
```

#### Existing container image version check
```
$ oc get po -n logging -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec -c {{$container.name}} {{$pod.metadata.name}} -n logging -- find /root/buildinfo -name Dockerfile-openshift* | grep -o logging.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

<<<<<<< HEAD
logging-elasticsearch-v3.7.52-1
….
logging-fluentd-v3.7.52-1
logging-kibana-v3.7.52-2
logging-auth-proxy-v3.7.52-1
=======
logging-curator-v3.7.61-2
logging-elasticsearch-v3.7.61-2
...
logging-fluentd-v3.7.61-2
logging-kibana-v3.7.61-2
logging-auth-proxy-v3.7.61-2

>>>>>>> 3.9
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

<<<<<<< HEAD
logging-curator-v3.7.61-2
logging-elasticsearch-v3.7.61-2
...
logging-fluentd-v3.7.61-2
logging-kibana-v3.7.61-2
logging-auth-proxy-v3.7.61-2
```

#### Upgrade the Cluster Metrics
=======
logging-curator-v3.9.40-2
logging-elasticsearch-v3.9.40-2
logging-fluentd-v3.9.40-2
…..
logging-kibana-v3.9.40-2
logging-auth-proxy-v3.9.40-2

```

### Upgrade the Cluster Metrics
>>>>>>> 3.9

*Commands*
```
$ ansible-playbook -i /etc/ansible/hosts ./playbooks/upgrade-to-latest-version.yml --tag always,metrics --skip-tags efk -e @vars/default.yml 
```

#### New container image version check
```
$ oc get po -n openshift-infra -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec {{$pod.metadata.name}} -- find /root/buildinfo -name Dockerfile-openshift* | grep -o metrics.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

<<<<<<< HEAD
metrics-cassandra-v3.7.61-11
metrics-hawkular-metrics-v3.7.61-11
metrics-heapster-v3.7.61-11
```

[Verify latest version of OCP](./verify-ocp-health.yml)
=======
metrics-cassandra-v3.9.40-11
metrics-hawkular-metrics-v3.9.40-11
metrics-heapster-v3.9.40-11

```

#### [Verify latest version of OCP](./verify-ocp-health.yml)
>>>>>>> 3.9
