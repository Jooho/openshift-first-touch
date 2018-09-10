OpenShift NFS Provisioner
-------------------------

## Description
This is not GA and still in [incubator project](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client). Therefore, please use it for test purpose.
This dynamic provisioner help you to create NFS PV via StorageClass. NFS provisioner is deploying NFS server on one of node in OpenShift. If you want to use your own external nfs server, please use NFS client provisioner.


## Pre-requisites

- Attach a disk to a VM.
- Make the disk to LVM and format it.
  ```
  export disk_name=vdc
  fdisk /dev/${disk_name} (n p enter enter enter t 8e w enter)
  pvcreate /dev/${disk_name}1
  vgcreate nfs-vg /dev/${disk_name}1
  lvcreate -l 50%VG -n nfs-lv nfs-vg
  mkfs.xfs /dev/nfs-vg/nfs-lv
  ```
- Create mount point and add it to fstab
  ```
  mkdir /exports-nfs
  echo "/dev/nfs-vg/nfs-lv  /exports-nfs xfs defaults 0 0" >> /etc/fstab
  ```

- Apply Selinux
  ```
  sudo chcon -Rt svirt_sandbox_file_t /exports-nfs/
  ```

- Add node label 
  ```
  ex) node hostname a.example.com

  oc label node a.example.com app=nfs-provisioner
  ```

## Default Variables

```
cat ./template.yaml
...
  name: NAMESPACE
  value: nfs-provisioner

  name: NFS_PATH
  value: "/export"

  name: HOST_PATH
  required: true
  value: "/exports-nfs"
```
*Note* 
NFS_PATH is hardcoded so can not changed [code](https://github.com/kubernetes-incubator/external-storage/blob/2db4446614545ef4ca87fd69ba81d24b1de390ca/nfs/cmd/nfs-provisioner/main.go)


## Deploy Variable
```
oc new-project nfs-provisioner
oc process -f  ./template.yml|oc create -f -

oc adm policy add-scc-to-user nfs-provisioner system:serviceaccount:nfs-provisioner:nfs-provisioner
oc adm policy add-cluster-role-to-user nfs-provisioner-runner system:serviceaccount:nfs-provisioner:nfs-provisioner

oc create -f storageclass.yaml

```



## Undeploy Steps
```
oc delete po/test-pod
oc delete pvc --all
oc delete sa nfs-provisioner
oc delete clusterrole nfs-provisioner-runner
oc delete clusterrolebindings "run-nfs-provisioner"
oc delete roles "leader-locking-nfs-provisioner"
oc delete rolebindings "leader-locking-nfs-provisioner"

oc delete scc nfs-provisioner
oc delete sc example-nfs
oc delete project nfs-provisioner 

```

## Test 
- Create PVC
```
$ oc create -f test-claim.yaml

$ oc get pvc
NAME           STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS   AGE
pvc/nfs        Bound     pvc-e8520b31-b52d-11e8-b0af-021a4a0ab215   1Mi        RWX           example-nfs    31m

```

- Deploy test pod
```
$ oc create -f test-pod.yaml

$ oc get pvc

$ oc get pod -o wide
NAME                               READY     STATUS    RESTARTS   AGE       IP            NODE
nfs-provisioner-3932204055-h9t8g   1/1       Running   0          7m        10.128.2.18   nfs.example.com
test-pod                           1/1       Running   0          7m        10.128.9.18   app.example.com

$ oc exec test-pod -- df -h
Filesystem                                                                                          Size  Used Avail Use% Mounted on
..
172.30.100.229:/export/pvc-01d79fd1-b53e-11e8-b0af-021a4a0ab215                                      25G   32M   25G   1% /mnt

$ oc exec test-pod -- touch /mnt/test.txt

# Check the VM that is running NFS provisioner
$ ssh nfs.example.com  -- ls -al /exports-nfs/pvc-01d79fd1-b53e-11e8-b0af-021a4a0ab215

total 0
drwxrwsrwx. 2 root       root       22 Sep 10 17:20 .
drwxrwxrwx. 3 root       root       121 Sep 10 17:10 ..
-rw-r--r--. 1 1000250000 root        0 Sep 10 17:20 test.txt

```

- Delete Pod & PVC
```
$ oc get pv,pvc
NAME                                          CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                               STORAGECLASS          REASON    AGE
pv/pvc-958fdfa0-b520-11e8-b0af-021a4a0ab215   1Mi        RWX           Delete          Bound       nfs-client-provisioner/test-claim   managed-nfs-storage             3h
pv/pvc-b9959f6d-b53f-11e8-b0af-021a4a0ab215   1Gi        RWX           Delete          Bound       nfs-provisioner/test-pvc            example-nfs                     2m

NAME           STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS   AGE
pvc/test-pvc   Bound     pvc-b9959f6d-b53f-11e8-b0af-021a4a0ab215   1Gi        RWX           example-nfs    2m


$ oc delete pod test-pod 

$ oc get pvc (still alive)
NAME       STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS   AGE
test-pvc   Bound     pvc-b9959f6d-b53f-11e8-b0af-021a4a0ab215   1Gi        RWX           example-nfs    2m


$ oc delete pvc test-pvc
$ oc get pv
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                               STORAGECLASS          REASON    AGE
.....

==> pv is also deleted and the direcotry in /export-nfs is also deleted.

$ ssh nfs.example.com  -- ls -al /exports-nfs
total 20
drwxrwxrwx.  2 root root   73 Sep 10 17:26 .
dr-xr-xr-x. 18 root root  243 Sep 10 14:16 ..
-rw-r--r--.  1 root root 9902 Sep 10 17:12 ganesha.log
-rw-------.  1 root root   36 Sep 10 17:06 nfs-provisioner.identity
-rw-------.  1 root root  648 Sep 10 17:26 vfs.conf





```
