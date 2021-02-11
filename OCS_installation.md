- [Introduction](#introduction)
- [Installing OCS Using Local Storage Devices](#installing-ocs-using-local-storage-devices)
- [Uninstalling OCS Using Local Storage Devices](#uninstalling-ocs-using-local-storage-devices)

## Introduction
Installing OpenShift Container Storage (OCS) on OCP 4.x.

In this document, there are steps applied to install OCS on OCP 4.3.  

References:
<br>
https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.3/html-single/deploying_openshift_container_storage/index
<br>
https://www.openshift.com/blog/blog-using-local-disks-on-vmware-with-openshift-container-storage

Topology:

* Installation server with following components
  - DNS server for OCP 4.3
  - Local Mirror for offline installation
* 1 load balancer for the control control plane (master nodes)
* 1 load balancer for the compute nodes (worker nodes)
* 1 Bootstrap node
* 3 control plane nodes (master nodes)
* 5 compute nodes (worker nodes)
* 18 500 GB VM Disks (Datastore)
* 3 10 GB VM Disks (Monitoring)

<br>


## Installing OCS Using Local Storage Devices

1. Find out which worker nodes you will use as OCS node. There should be at least 3 nodes available for OCS deployment

    ```
    $ oc get nodes
        NAME                                         STATUS   ROLES    AGE   VERSION
        <master 1>.<OCPDomainPrefix>.<BaseDomain>   Ready    master   21d   v1.16.2
        <master 2>.<OCPDomainPrefix>.<BaseDomain>   Ready    master   21d   v1.16.2
        <master 3>.<OCPDomainPrefix>.<BaseDomain>   Ready    master   21d   v1.16.2
        <worker 1>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 2>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 3>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 4>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
        <worker 5>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    ```

2. Add label of "cluster.ocs.openshift.io/openshift-storage=''" to the worker nodes you will add local devices

    ```
    $ oc label nodes <worker 3>.<OCPDomainPrefix>.<BaseDomain> cluster.ocs.openshift.io/openshift-storage=''
    node/<worker 3>.<OCPDomainPrefix>.<BaseDomain> labeled
    $ oc label nodes <worker 4>.<OCPDomainPrefix>.<BaseDomain> cluster.ocs.openshift.io/openshift-storage=''
    node/<worker 4>.<OCPDomainPrefix>.<BaseDomain> labeled
    $ oc label nodes <worker 5>.<OCPDomainPrefix>.<BaseDomain> cluster.ocs.openshift.io/openshift-storage=''
    node/<worker 5>.<OCPDomainPrefix>.<BaseDomain> labeled    

    $ oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
    NAME                                         STATUS   ROLES    AGE   VERSION
    <worker 3>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    <worker 4>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    <worker 5>.<OCPDomainPrefix>.<BaseDomain>   Ready    worker   21d   v1.16.2
    ```

3. Create a namespace for OCS

    - Click Administration → Namespaces in the left pane of the OpenShift Web Console.
    - Click Create Namespace.
    - In the Create Namespace dialog box, enter **openshift-storage** for Name and **openshift.io/cluster-monitoring=true** for Labels. This label is required to get the dashboards.
    - Select No restrictions option for Default Network Policy.
    - Click Create.

4. Create a namespace for LocalStorage

    - Click Administration → Namespaces in the left pane of the OpenShift Web Console.
    - Click Create Namespace.
    - In the Create Namespace dialog box, enter **local-storage** for Name.
    - Select No restrictions option for Default Network Policy.
    - Click Create.

5. Install OCS operator using Operator HUB

    - Click Operators → OperatorHub in the left pane of the OpenShift Web Console.
    - Find and click install for OpenShift Container Storage Operator page.
    - Pick namespace you created above (openshift-storage), pick appropriate update channel (stable-4.3) and pick Automatic Approval strategy
    - Click subscribe

6. Install Local Storage operator using Operator HUB

    - Click Operators → OperatorHub in the left pane of the OpenShift Web Console.
    - Find and click install for Local Storage Operator page.
    - Pick namespace you created above (local-storage), pick appropriate update channel (stable-4.3) and pick Automatic Approval strategy
    - Click subscribe

7. Login to each worker nodes which you have storage's asssigned and note their disk id's.

    ```
        [core@<worker 3> ~]$ ls -latr /dev/disk/by-id/ | grep wwn | grep -v sda
        lrwxrwxrwx. 1 root root   9 Jun 15 06:14 wwn-0x6000c29bbfb25bcba881692e9e016606 -> ../../sdd
        lrwxrwxrwx. 1 root root   9 Jun 15 06:14 wwn-0x6000c299060c007effb11ed17d1e364d -> ../../sde
        lrwxrwxrwx. 1 root root   9 Jun 15 06:14 wwn-0x6000c2989ff5753dfb6faae60ddf88c0 -> ../../sdf
        lrwxrwxrwx. 1 root root   9 Jun 15 06:14 wwn-0x6000c29508154cbe96e70a369ec1cfa8 -> ../../sdb
        lrwxrwxrwx. 1 root root   9 Jun 15 06:15 wwn-0x6000c2990be86fa35cac3a54acc1fee0 -> ../../sdc
        lrwxrwxrwx. 1 root root   9 Jun 15 06:15 wwn-0x6000c29a7a8dd36243413c4103aee40b -> ../../sdg
        lrwxrwxrwx. 1 root root   9 Jun 15 06:16 wwn-0x6000c29848cfd6937a4796964b4d2ee4 -> ../../sdh



        [core@<worker 4> ~]$ ls -latr /dev/disk/by-id/ | grep wwn | grep -v sda
        lrwxrwxrwx. 1 root root   9 Jun 10 19:45 wwn-0x6000c294c867fde5744a2039493c34ed -> ../../sdd
        lrwxrwxrwx. 1 root root   9 Jun 10 19:45 wwn-0x6000c293fd080ed660a1f1884f1dd06f -> ../../sde
        lrwxrwxrwx. 1 root root   9 Jun 10 19:45 wwn-0x6000c295da6440bba31777f93d4910d5 -> ../../sdb
        lrwxrwxrwx. 1 root root   9 Jun 10 19:45 wwn-0x6000c297977ae57ccdb65487a05bdb20 -> ../../sdc
        lrwxrwxrwx. 1 root root   9 Jun 10 19:46 wwn-0x6000c29bc6bb231463c8887db7b2d50d -> ../../sdh
        lrwxrwxrwx. 1 root root   9 Jun 10 19:46 wwn-0x6000c2914d6e7ad957d5c517fdb92982 -> ../../sdf
        lrwxrwxrwx. 1 root root   9 Jun 10 19:46 wwn-0x6000c29eebe188d67f6ce083461bb950 -> ../../sdg



        [core@<worker 5> ~]$ ls -latr /dev/disk/by-id/ | grep wwn | grep -v sda
        lrwxrwxrwx. 1 root root   9 Jun 10 19:48 wwn-0x6000c290232200fea7f56c4a0500a1d6 -> ../../sdc
        lrwxrwxrwx. 1 root root   9 Jun 10 19:48 wwn-0x6000c297b39f9f29c275a630cd4c566b -> ../../sde
        lrwxrwxrwx. 1 root root   9 Jun 10 19:48 wwn-0x6000c292faef838b3f59f94f10fc4e0a -> ../../sdd
        lrwxrwxrwx. 1 root root   9 Jun 10 19:48 wwn-0x6000c2913dba7e4f083655a4076a9315 -> ../../sdf
        lrwxrwxrwx. 1 root root   9 Jun 10 19:49 wwn-0x6000c2979e52a72c6245a6b9f2a1d43b -> ../../sdg
        lrwxrwxrwx. 1 root root   9 Jun 10 19:49 wwn-0x6000c29c771fa362953192ecb293f308 -> ../../sdb
        lrwxrwxrwx. 1 root root   9 Jun 10 19:49 wwn-0x6000c2964a5b11b0ebb6b87e920db6d3 -> ../../sdh

    ```

8. Create local storage block Volumes.

    - Create following yaml file with using disk id numbers gathered. **DO NOT PUT Monitoring Disks here**
    ```
        $ vi local-storage-block.yaml
        apiVersion: local.storage.openshift.io/v1
        kind: LocalVolume
        metadata:
          name: local-block
          namespace: local-storage
        spec:
          nodeSelector:
            nodeSelectorTerms:
            - matchExpressions:
                - key: cluster.ocs.openshift.io/openshift-storage
                  operator: In
                  values:
                  - ""
          storageClassDevices:
            - storageClassName: localblock
              volumeMode: Block
              devicePaths:
                - /dev/disk/by-id/wwn-0x6000c2990be86fa35cac3a54acc1fee0
                - /dev/disk/by-id/wwn-0x6000c29508154cbe96e70a369ec1cfa8
                - /dev/disk/by-id/wwn-0x6000c299060c007effb11ed17d1e364d
                - /dev/disk/by-id/wwn-0x6000c2989ff5753dfb6faae60ddf88c0
                - /dev/disk/by-id/wwn-0x6000c29a7a8dd36243413c4103aee40b
                - /dev/disk/by-id/wwn-0x6000c29bbfb25bcba881692e9e016606
                - /dev/disk/by-id/wwn-0x6000c293fd080ed660a1f1884f1dd06f
                - /dev/disk/by-id/wwn-0x6000c297977ae57ccdb65487a05bdb20
                - /dev/disk/by-id/wwn-0x6000c295da6440bba31777f93d4910d5
                - /dev/disk/by-id/wwn-0x6000c2914d6e7ad957d5c517fdb92982
                - /dev/disk/by-id/wwn-0x6000c294c867fde5744a2039493c34ed
                - /dev/disk/by-id/wwn-0x6000c29eebe188d67f6ce083461bb950
                - /dev/disk/by-id/wwn-0x6000c297b39f9f29c275a630cd4c566b
                - /dev/disk/by-id/wwn-0x6000c29c771fa362953192ecb293f308
                - /dev/disk/by-id/wwn-0x6000c290232200fea7f56c4a0500a1d6
                - /dev/disk/by-id/wwn-0x6000c292faef838b3f59f94f10fc4e0a
                - /dev/disk/by-id/wwn-0x6000c2913dba7e4f083655a4076a9315
                - /dev/disk/by-id/wwn-0x6000c2964a5b11b0ebb6b87e920db6d3
                - /dev/disk/by-id/wwn-0x6000c2979e52a72c6245a6b9f2a1d43b
    ```
    - Run create command using oc command

    ```
        $ oc create -f local-storage-block.yaml
        localvolume.local.storage.openshift.io/local-block created
    ```

    - Make sure all pods are running and active
    ```
    $ oc -n local-storage get pods
    NAME                                      READY   STATUS    RESTARTS   AGE
    local-block-local-diskmaker-45pdv         1/1     Running   0          14s
    local-block-local-diskmaker-fk94d         1/1     Running   0          14s
    local-block-local-diskmaker-pnnns         1/1     Running   0          14s
    local-block-local-provisioner-r225h       1/1     Running   0          14s
    local-block-local-provisioner-tlnh7       1/1     Running   0          14s
    local-block-local-provisioner-zhmdq       1/1     Running   0          14s
    local-storage-operator-6f74747555-bl7hq   1/1     Running   0          3m49s
    ```

9. Create OCS service

    - Navigate to Operators → Installed Operators → Openshift Container Storage
    - Go to Storage Cluster tab and click new
    - Pick your 3 worker nodes from the list
    - Pick storage class as localblock
    - Pick OCS service capacity as 500MiB (This should be the disk size of your each block)
    - Click create

10. Creat a new local-files storage type and added 10 Gib from each worker as type filesystems

    - Create following yaml file with using disk id numbers for monitoring (10Gb disks).

    ```
    vi local-storage-fs.yaml
    apiVersion: local.storage.openshift.io/v1
    kind: LocalVolume
    metadata:
      name: local-files
      namespace: local-storage
    spec:
      nodeSelector:
        nodeSelectorTerms:
        - matchExpressions:
            - key: cluster.ocs.openshift.io/openshift-storage
              operator: In
              values:
              - ""
      storageClassDevices:
        - storageClassName: localfile
          volumeMode: Filesystem
          devicePaths:
            - /dev/disk/by-id/wwn-0x6000c2964a5b11b0ebb6b87e920db6d3
            - /dev/disk/by-id/wwn-0x6000c29bc6bb231463c8887db7b2d50d
            - /dev/disk/by-id/wwn-0x6000c29848cfd6937a4796964b4d2ee4
    ```

    - Run create command using oc command

    ```
        $ oc apply -f local-storage-fs.yaml
        localvolume.local.storage.openshift.io/local-files created  
    ```

11. Delete and recreate PVC for monitoring pods if defult PVC are in still pending state. It happens when pvc was created for block but you have filesystem storage for monitoring

    - Copy old PVC for one of monitoring pods.
    - Delete old pvc's.
    - Create one by one with changing monitoring pod suffix with "a", "b" and "c"
    <br>**DO NOT FORGET TO CHANGE "storageClassName: localfile" !!!**<br>

    ```
        one example:
        kind: PersistentVolumeClaim
        apiVersion: v1
        metadata:
          selfLink: /api/v1/namespaces/openshift-storage/persistentvolumeclaims/rook-ceph-mon-c
          resourceVersion: '9494332'
          name: rook-ceph-mon-c
          uid: 497991d8-0ba4-41b7-b001-95dd523994d3
          creationTimestamp: '2020-06-05T08:49:39Z'
          namespace: openshift-storage
          ownerReferences:
            - apiVersion: ceph.rook.io/v1
              kind: CephCluster
              name: ocs-storagecluster-cephcluster
              uid: b50c1189-f3eb-46aa-bf7b-4388882521e3
              blockOwnerDeletion: true
          finalizers:
            - kubernetes.io/pvc-protection
          labels:
            app: rook-ceph-mon
            rook-version: 4.3-40.699b9ce7.ocs_4.3
            ceph-version: 14.2.4-125
            mon_cluster: openshift-storage
            mon_canary: 'true'
            mon: c
            pvc_name: rook-ceph-mon-c
            ceph_daemon_id: c
            rook_cluster: openshift-storage
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 10Gi
          storageClassName: localfile
          volumeMode: Filesystem

    ```


## Uninstalling OCS Using Local Storage Devices

1) Follow the steps mentioned at redhat official documentation

    https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.3/html-single/deploying_openshift_container_storage/index#uninstalling-openshift-container-storage_rhocs

2) Delete hung CRDs if exists.
    - List hung CRD's
    ```
    $ oc get crd | grep -i ceph
    cephclusters.ceph.rook.io                                   2020-06-04T13:11:50Z
    $ oc get crd | grep -i localvolumes
    localvolumes.local.storage.openshift.io                     2020-06-03T09:35:37Z
    ```

    - Set finalizer to none

    ```
    $ oc patch crd/cephclusters.ceph.rook.io -p '{"metadata":{"finalizers":[]}}' --type=merge
    $ oc patch crd/localvolumes.local.storage.openshift.io  -p '{"metadata":{"finalizers":[]}}' --type=merge
    ```

3) Login to each worker node with OCS label and delete remaining links if exists if you have following error:

    ```error creating symlink /mnt/local-storage/localblock/sdc: <nil>```

    ```
    # cd /mnt/local-storage/localblock/
    # ls
    sdb  sdc  sdd  sde  sdf  sdg  sdh
    # rm -Rf *
    ```
