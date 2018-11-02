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
