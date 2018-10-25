Verify OpenShift Health
-----------------------

##Export Environment Variables
```
export API_SERVER=openshift.example.com:8443
```


## Deploy Test Application for Verification

*Commands*
```
$ oc new-project test
$ oc new-app https://github.com/gshipley/smoke.git
$ oc expose service smoke --port=8080
$ curl smoke<subdomain>
Welcome to the OpenShift 3 Roadshow Smoke Test Application

# Keep checking the response
$ while true; do  echo $(date) >> /tmp/test.out ;curl smoke.<SUBDOMAIN> >> /tmp/test.out; sleep 2; done&
```

## Useful Commands

### API Server
```
# Health_check
$ curl $API_SERVER/healthz
# Kubernetes api version
$ curl $API_SERVER/version
# Openshift version
$ curl $API_SERVER/version/openshift
```

### Registry
```
# Image Version Check
$ oc get -n default dc/docker-registry -o json | grep \"image\"
```

### Router
```
# Image Version Check
$ oc get -n default dc/router -o json | grep \"image\"
```

### EFK
```
# Image Version Check
$ oc get po -n logging -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec -c {{$container.name}} {{$pod.metadata.name}} -n logging -- find /root/buildinfo -name Dockerfile-openshift* | grep -o logging.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -

# Health Check ElasticSearch (inside ES)
$curl -s --key /etc/elasticsearch/secret/admin-key --cert /etc/elasticsearch/secret/admin-cert --cacert /etc/elasticsearch/secret/admin-ca "https://localhost:9200/_cat/health?v"

$curl -s --key /etc/elasticsearch/secret/admin-key --cert /etc/elasticsearch/secret/admin-cert --cacert /etc/elasticsearch/secret/admin-ca "https://localhost:9200/_cat/nodes?v"
```

### Metrics
```
# Image Version Check
$oc get po -n openshift-infra -o 'go-template={{range $pod := .items}}{{if eq $pod.status.phase "Running"}}{{range $container := $pod.spec.containers}}oc exec {{$pod.metadata.name}} -- find /root/buildinfo -name Dockerfile-openshift* | grep -o metrics.* {{"\n"}}{{end}}{{end}}{{end}}' | bash -
```

