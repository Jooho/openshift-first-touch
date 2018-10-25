Openshift NFS client provisioner
-------------------------------

## Description
This is not GA and still in [incubator project](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client). Therefore, please use it for test purpose.
This dynamic provisioner help you to create NFS PV via StorageClass. NFS client provisioner is using external NFS server. If you want to use your own containerized nfs server on openshift, please use NFS provisioner.

If you want to install NFS server, you can use this [NFS Server installation ansible playbook](https://github.com/Jooho/ansible-cheat-sheet/tree/master/ansible-playbooks/ansible-playbook-nfs-server).


## Default Variables
```
- displayName: Namespace
  name: NAMESPACE
  required: true
  value: nfs-client-provisioner
- displayName: NFS_PATH
  name: NFS_PATH
  required: true
  value: "/exports"
- displayName: NFS_SERVER
  name: NFS_SERVER
  required: true
  value: "example.com"

```


## Deploy Steps
```
oc new-project nfs-client-provisioner
oc process -f deployment.yaml -p NFS_SERVER=$NFS_SERVER_HOST_IP -p NFS_PATH=$NFS_FODLER_PATH -p NAMESPACE=nfs-client-provisioner |oc create -f -


(ex)
oc process -f deployment.yaml -p NFS_SERVER=nfs.exmale.com -p NFS_PATH=/exports-nfs -p NAMESPACE=nfs-client-provisioner |oc create -f -

oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:nfs-client-provisioner:nfs-client-provisioner

oc create -f storageClass.yaml 

oc patch sc managed-nfs-storage -p   '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true" }}}'
```


## Undeploy Steps
```

oc delete serviceaccounts "nfs-client-provisioner" 
oc delete clusterroles.rbac.authorization.k8s.io "nfs-client-provisioner-runner"
oc delete clusterrolebindings.rbac.authorization.k8s.io "run-nfs-client-provisioner" 
oc delete roles.rbac.authorization.k8s.io "leader-locking-nfs-client-provisioner" 
oc delete rolebindings.rbac.authorization.k8s.io "leader-locking-nfs-client-provisioner"
oc delete deployments.extensions "nfs-client-provisioner" 
oc delete project nfs-client-provisioner

```



## Test 
- Create PVC
```
$ oc create -f test-claim.yaml

$ oc get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS          AGE
test-claim   Bound     pvc-6c795823-b54a-11e8-b0af-021a4a0ab215   1Mi        RWX           managed-nfs-storage   4s

```

- Deploy test pod
```
$ oc create -f test-pod.yaml

$ oc get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS          AGE
test-claim   Bound     pvc-6c795823-b54a-11e8-b0af-021a4a0ab215   1Mi        RWX           managed-nfs-storage   7m
test-pvc     Bound     pvc-07ade58e-b54b-11e8-b0af-021a4a0ab215   1Gi        RWX           managed-nfs-storage   3m


$ oc get pod -o wide
NAME                               READY     STATUS    RESTARTS   AGE       IP            NODE
nfs-client-provisioner-3143141697-f8gxh   1/1       Running   0          5h        10.128.0.12   dhcp181-50.gsslab.rdu2.redhat.com
test-pod                                  1/1       Running   0          46s       10.128.9.19   vm4.gsslab.rdu2.redhat.com


$ oc exec test-pod -- df -h
Filesystem                                                                                          Size  Used Avail Use% Mounted on
..
nfs.example.com:/exports-nfs/nfs-client-provisioner-test-pvc-pvc-07ade58e-b54b-11e8-b0af-021a4a0ab215   50G  233M   50G   1% /mnt


$ oc exec test-pod -- touch /mnt/test.txt

# Check the VM that is running NFS provisioner
$ ssh nfs.example.com  -- ls -al /exports-nfs/nfs-client-provisioner-test-pvc-pvc-07ade58e-b54b-11e8-b0af-021a4a0ab215
drwxrwxrwx.  2 root       root        18 Sep 10 18:56 .
drwxrwxrwx. 10 root       root      4096 Sep 10 18:54 ..
-rw-r--r--.  1 1000190000 nfsnobody    5 Sep 10 18:56 test.txt

```

- Delete Pod & PVC
```
$ oc get pv,pvc
NAME                                          CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                               STORAGECLASS          REASON    AGE
pv/pvc-6c795823-b54a-11e8-b0af-021a4a0ab215   1Mi        RWX           Delete          Bound       nfs-client-provisioner/test-claim   managed-nfs-storage             18m
pv/pvc-6cdaa0aa-b54c-11e8-b0af-021a4a0ab215   1Gi        RWX           Delete          Bound       nfs-client-provisioner/test-pvc     managed-nfs-storage             4m

NAME             STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS          AGE
pvc/test-claim   Bound     pvc-6c795823-b54a-11e8-b0af-021a4a0ab215   1Mi        RWX           managed-nfs-storage   18m
pvc/test-pvc     Bound     pvc-6cdaa0aa-b54c-11e8-b0af-021a4a0ab215   1Gi        RWX           managed-nfs-storage   4m

$ oc delete pod test-pod 

$ oc get pvc (still alive)
NAME         STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS          AGE
test-claim   Bound     pvc-6c795823-b54a-11e8-b0af-021a4a0ab215   1Mi        RWX           managed-nfs-storage   19m
test-pvc     Bound     pvc-6cdaa0aa-b54c-11e8-b0af-021a4a0ab215   1Gi        RWX           managed-nfs-storage   5m


$ oc delete pvc test-pvc

$ oc get pv
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                               STORAGECLASS          REASON    AGE
.....

==> pv is also deleted and the direcotry in /export-nfs is also deleted.

$ ssh nfs.example.com  -- ls -al /exports-nfs
-rw-r--r--.  1 root root 46901 Sep 10 12:56 ganesha.log
-rw-------.  1 root root    36 Sep 10 09:51 nfs-provisioner.identity
-rw-------.  1 root root   648 Sep 10 12:56 vfs.conf

```
