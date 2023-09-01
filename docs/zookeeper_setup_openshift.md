# Setting up Zookeeper

This document describes how to setup ZooKeeper in OpenShift environment.

Zookeeper installation is available in two options:
1. [Quick start](#quick-start) - just run it quickly and ask no questions
1. [Advanced setup](#advanced-setup) - setup internal details, such as storage class, replicas number, etc 

During ZooKeeper installation the following items are created/configured:
1. [OPTIONAL] Create separate namespace to run Zookeeper in 
1. Create k8s resources (optionally, within namespace):
  * [Service][k8sdoc_service_main] - used to provide central access point to Zookeeper
  * [Headless Service][k8sdoc_service_headless] - used to provide DNS namings
  * [Disruption Balance][k8sdoc_disruption_balance] - used to specify max number of offline pods
  * [OPTIONAL] [Storage Class][k8sdoc_storage_class] - used to specify storage class to be used by Zookeeper for data storage
  * [Stateful Set][k8sdoc_statefulset] - used to manage and scale sets of pods

## Quick start
Quick start is represented in two flavors:
1. With persistent volume - good for performance. File are located in [deploy/zookeeper/quick-start-persistent-volume][quickstart_persistent] 
1. With local [`emptyDir`][k8sdoc_emptydir] storage - good for standalone local run, however has to true persistence. \
Files are located in [deploy/zookeeper/quick-start-volume-emptyDir][quickstart_emptydir] 

Each quick start flavor provides the following installation options:
1. 1-node Zookeeper cluster (**zookeeper-1-node-openshift**). No failover provided.
1. 3-node Zookeeper cluster (**zookeeper-3-node-openshift**). Failover provided.

In case you'd like to have better performance, we recommend to go with [deploy/zookeeper/quick-start-persistent-volume][quickstart_persistent] persistent storage.
In case of local test, you may prefer to go with [deploy/zookeeper/quick-start-volume-emptyDir][quickstart_emptydir] `emptyDir`.

### Manual Installation
In case you'd like to deploy Zookeeper manually, the following steps should be performed:

### Namespace
Create **namespace**
```bash
kubectl create namespace zoo1ns
```
 
### Zookeeper
Deploy Zookeeper into this namespace
```bash
kubectl apply -f zookeeper-1-node.yaml -n zoo1ns
```

Now Zookeeper should be up and running. Let's [explore Zookeeper cluster](#explore-zookeeper-cluster).

**IMPORTANT** quick-start zookeeper installation are for test purposes mainly.  
For fine-tuned Zookeeper setup please refer to [advanced setup](#advanced-setup) options.  

## Advanced setup
Advanced files are are located in [deploy/zookeeper/advanced][zookeeper-advanced] folder. 
All resources are separated into different files so it is easy to modify them and setup required options.  

Advanced setup is available in two options:
1. With [persistent volume][k8sdoc_persistent_volume]
1. With [emptyDir volume][k8sdoc_emptydir]

Each of these options have both `create` and `delete` scripts provided
1. Persistent volume  [create][zookeeper-persistent-volume-create.sh] and [delete][zookeeper-persistent-volume-delete.sh] scripts
1. EmptyDir volume  [create][zookeeper-volume-emptyDir-create.sh] and [delete][zookeeper-volume-emptyDir-delete.sh] scripts

Step-by-step explanations:

### Namespace
Create **namespace** in which all the rest resources would be created
```bash
kubectl create namespace zoons
```
 
### Zookeeper Service
Create service. This service provides DNS name for client access to all Zookeeper nodes.
```bash
kubectl apply -f 01-service-client-access.yaml -n zoons
```
Should have as a result
```text
service/zookeeper created
```

### Zookeeper Headless Service
Create headless service. This service provides DNS names for all Zookeeper nodes
```bash
kubectl apply -f 02-headless-service.yaml -n zoons
```
Should have as a result
```text
service/zookeeper-nodes created
```

### Disruption Budget
Create budget. Disruption Budget instructs k8s on how many offline Zookeeper nodes can be at any time
```bash
kubectl apply -f 03-pod-disruption-budget.yaml -n zoons
``` 
Should have as a result
```text
poddisruptionbudget.policy/zookeeper-pod-distribution-budget created
```

### Storage Class
This part is not that straightforward and may require communication with k8s instance administrator.

First of all, we need to decide, whether Zookeeper would use [Persistent Volume][k8sdoc_persistent_volume] 
as a storage or just stick to more simple [Volume][k8sdoc_volume] (In doc [emptyDir][k8sdoc_emptydir] type is used)

In case we'd prefer to stick with simpler solution and go with [Volume of type emptyDir][k8sdoc_emptydir], 
we need to go with **emptyDir StatefulSet config** [05-stateful-set-volume-emptyDir-openshift.yaml][05-stateful-set-volume-emptyDir-openshift.yaml] as described in next [Stateful Set unit](#stateful-set). Just move to [it](#stateful-set).

In case we'd prefer to go with [Persistent Volume][k8sdoc_persistent_volume] storage, we need to go 
with **Persistent Volume StatefulSet config** [05-stateful-set-persistent-volume-openshift.yaml][05-stateful-set-persistent-volume-openshift.yaml]

In OpenShift, we can leverage [Local Storage Operator](https://github.com/openshift/local-storage-operator).
After login into OpenShift, select **Operators > OperatorHub** on the left.
In the "Filter by keyword" input box, type ```local``` and click the **Local Storage** tile as follows:\
![Local Storage Operator](./img/local-storage-tile.png)\
Click the **install** button\
![Install Local Storage Operator](./img/install-local-storage-operator.png)\
and scroll down to the bottom on the next page. At the end, click the **install** button to get it going. After a while, you will see the following means the installation is completed:\
![Local Storage Operator installed](./img/local-storage-operator-installed.png)

Assume that your OpenShift have two extra disks on each worker node. The following [LocalVolume Custom Resource][04-local-volume-local-sc.yaml] can be used by the Local Storage Operator to create **local-sc** storage class.
```yaml
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-disks"
  namespace: "openshift-local-storage"
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker0.<domain>.cp.fyre.ibm.com
          - worker1.<domain>.cp.fyre.ibm.com
          - worker2.<domain>.cp.fyre.ibm.com
  storageClassDevices:
    - storageClassName: "local-sc"
      volumeMode: Filesystem
      fsType: xfs
      devicePaths:
        - /dev/vdb
        - /dev/vdc
```
Note: You need to change &lt;domain> to match your cluster's settings. Change /dev/vdb and /dev/vdc to match your node's configuration if needed.

After you finish the required changes, run the following commands to make the **local-sc** storage class available. The newly created storage class will be the default one, and can be selected by **Persistent Volume StatefulSet config** [05-stateful-set-persistent-volume-openshift.yaml][05-stateful-set-persistent-volume-openshift.yaml]
```bash
kubectl apply -f ../deploy/zookeeper/advanced/04-local-volume-local-sc.yaml

kubectl patch storageclass local-sc -n openshift-local-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Stateful Set
Edit **StatefulSet manifest with emptyDir** [05-stateful-set-volume-emptyDir-openshift.yaml][05-stateful-set-volume-emptyDir-openshift.yaml] 
and/or **StatefulSet manifest with Persistent Volume** [05-stateful-set-persistent-volume-openshift.yaml][05-stateful-set-persistent-volume-openshift.yaml] 
according to your Storage Preferences.

In case we'd go with [Volume of type emptyDir][k8sdoc_emptydir], ensure `.spec.template.spec.containers.volumes` is in place and look like the following:
```yaml
      volumes:
      - name: datadir-volume
        emptyDir:
          medium: "" #accepted values:  empty str (means node's default medium) or Memory
          sizeLimit: 1Gi
```

In case we'd go with **Persistent Volume** storage, ensure `.spec.volumeClaimTemplates` is there as follows.
```yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir-volume
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 25Gi
```
Note: There is no **storageClassName** specified here. This implies the default storage class will be used. If you need to use other storage class, please specify it at the same level as **spec**.

As `.yaml` file is ready, just apply it with `kubectl`
```bash
kubectl apply -f 05-stateful-set-<persistent-volume|volume-emptyDir>-openshift.yaml -n zoons
```
Should have as a result
```text
statefulset.apps/zookeeper created
```

Now we can take a look into Zookeeper cluster deployed in k8s:

## Explore Zookeeper cluster

### DNS names
We are expecting to have ZooKeeper cluster of 3 pods inside `zoons` namespace, named as:
```text
zookeeper-0
zookeeper-1
zookeeper-2
```
Those pods are expected to have short DNS names as:

```text
zookeeper-0.zookeepers.zoons
zookeeper-1.zookeepers.zoons
zookeeper-2.zookeepers.zoons
```

where `zookeepers` is name of [Zookeeper headless service](#zookeeper-headless-service) and `zoons` is name of [Zookeeper namespace](#namespace).

and full DNS names (FQDN) as:
```text
zookeeper-0.zookeepers.zoons.svc.cluster.local
zookeeper-1.zookeepers.zoons.svc.cluster.local
zookeeper-2.zookeepers.zoons.svc.cluster.local
```

### Resources

List pods in Zookeeper's namespace
```bash
kubectl get pod -n zoons
```

Expected output is like the following
```text
NAME             READY   STATUS    RESTARTS   AGE
zookeeper-0      1/1     Running   0          9m2s
zookeeper-1      1/1     Running   0          9m2s
zookeeper-2      1/1     Running   0          9m2s
```

List services
```bash
kubectl get service -n zoons
```

Expected output is like the following
```text
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
zookeeper              ClusterIP   10.108.36.44   <none>        2181/TCP                     168m
zookeepers             ClusterIP   None           <none>        2888/TCP,3888/TCP            31m
```

List statefulsets
```bash
kubectl get statefulset -n zoons
```

Expected output is like the following
```text
NAME            READY   AGE
zookeepers      3/3     10m
```

In case all looks fine Zookeeper cluster is up and running



[k8sdoc_service_main]: https://kubernetes.io/docs/concepts/services-networking/service/
[k8sdoc_service_headless]: https://kubernetes.io/docs/concepts/services-networking/service/#headless-services
[k8sdoc_disruption_balance]: https://kubernetes.io/docs/concepts/workloads/pods/disruptions/
[k8sdoc_storage_class]: https://kubernetes.io/docs/concepts/storage/storage-classes/
[k8sdoc_statefulset]: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
[k8sdoc_volume]: https://kubernetes.io/docs/concepts/storage/volumes
[k8sdoc_emptydir]: https://kubernetes.io/docs/concepts/storage/volumes/#emptydir
[k8sdoc_persistent_volume]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/
[k8sdoc_persistent_volume_claim]: https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims
[k8sdocs_dynamic_provisioning]: https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/

[quickstart_persistent]: https://github.com/Altinity/clickhouse-operator/tree/master/deploy/zookeeper/quick-start-persistent-volume
[quickstart_emptydir]: https://github.com/Altinity/clickhouse-operator/tree/master/deploy/zookeeper/quick-start-volume-emptyDir

[zookeeper-advanced]: https://github.com/Altinity/clickhouse-operator/tree/master/deploy/zookeeper/advanced
[zookeeper-persistent-volume-create.sh]: https://github.com/Altinity/clickhouse-operator/blob/master/deploy/zookeeper/advanced/zookeeper-persistent-volume-create.sh
[zookeeper-persistent-volume-delete.sh]: https://github.com/Altinity/clickhouse-operator/blob/master/deploy/zookeeper/advanced/zookeeper-persistent-volume-delete.sh
[zookeeper-volume-emptyDir-create.sh]: https://github.com/Altinity/clickhouse-operator/blob/master/deploy/zookeeper/advanced/zookeeper-volume-emptyDir-create.sh
[zookeeper-volume-emptyDir-delete.sh]: https://github.com/Altinity/clickhouse-operator/blob/master/deploy/zookeeper/advanced/zookeeper-volume-emptyDir-delete.sh
[04-local-volume-local-sc.yaml]: ../deploy/zookeeper/advanced/04-local-volume-local-sc.yaml
[05-stateful-set-volume-emptyDir-openshift.yaml]: ../deploy/zookeeper/advanced/05-stateful-set-volume-emptyDir-openshift.yaml
[05-stateful-set-persistent-volume-openshift.yaml]: ../deploy/zookeeper/advanced/05-stateful-set-persistent-volume-openshift.yaml