IMPORTANT
The following article is not made to remove only certain components of ODF, but to uninstall all of ODF

Use the steps in this section to uninstall OpenShift Data Foundation.

Uninstall Annotations
Annotations on the Storage Cluster are used to change the behavior of the uninstall process. To define the uninstall behavior, the following two annotations have been introduced in the storage cluster:

uninstall.ocs.openshift.io/cleanup-policy: delete
uninstall.ocs.openshift.io/mode: graceful
These annotations should be fully understood before applying them to an ocs cluster. The below table provides information on the different values that can be used with these annotations:

Uninstall annotations descriptions
Raw
cleanup-policy = delete 
Default value. Rook cleans up the physical drives and the DataDirHostPath

Raw
cleanup-policy = retain
Rook does not clean up the physical drives and the DataDirHostPath

Raw
mode=graceful
Rook and NooBaa pauses the uninstall process until the administrator/user removes the Persistent Volume Claims (PVCs) and Object Bucket Claims (OBCs)

Raw
mode=forced
Rook and NooBaa proceed with uninstalling even if the PVCs/OBCs provisioned using Rook and NooBaa exist respectively.

Edit the value of the annotation to change the cleanup policy or the uninstall mode.

Raw
$ oc annotate storagecluster -n openshift-storage ocs-storagecluster uninstall.ocs.openshift.io/cleanup-policy="<CLEANUP_POLICY>" --overwrite
Raw
$  oc annotate storagecluster -n openshift-storage ocs-storagecluster uninstall.ocs.openshift.io/mode="<MODE>" --overwrite
The expected output for both commands:

Raw
storagecluster.ocs.openshift.io/ocs-storagecluster annotated
Prerequisites

Ensure that the OpenShift Data Foundation cluster is in a healthy state.
The uninstall process can fail when some of the pods are not terminated successfully due to insufficient resources or nodes.
In case the cluster is in an unhealthy state, contact Red Hat Customer Support before uninstalling OpenShift Data Foundation.
Ensure that applications are not consuming persistent volume claims (PVCs) or object bucket claims (OBCs) using the storage classes provided by OpenShift Data Foundation.
If any custom resources (such as custom storage classes, cephblockpools) were created by the admin, they must be deleted by the admin after removing the resources which consumed them.
Procedure

Delete the volume snapshots that are using OpenShift Data Foundation.

List the volume snapshots from all the namespaces.
Raw
$ oc get volumesnapshot --all-namespaces
From the output of the previous command, identify and delete the volume snapshots that are using OpenShift Data Foundation.
Raw
$ oc delete volumesnapshot <VOLUME-SNAPSHOT-NAME> -n <NAMESPACE>
<VOLUME-SNAPSHOT-NAME> Is the name of the volume snapshot
<NAMESPACE> Is the project namespace

Delete PVCs and OBCs that are using OpenShift Data Foundation. In the default uninstall mode (graceful), the uninstaller waits till all the PVCs and OBCs that use OpenShift Data Foundation are
deleted. If you want to delete the Storage Cluster without deleting the PVCs beforehand, you may set the uninstall mode annotation to forced and skip this step. Doing this results in orphan PVCs and OBCs in the system.

Delete OpenShift Container Platform monitoring stack PVCs using OpenShift Data Foundation.
For more information, see, Removing monitoring stack from OpenShift Data Foundation.
Delete OpenShift Container Platform Registry PVCs using OpenShift Data Foundation.
For more information, see, Removing OpenShift Container Platform registry from OpenShift Data Foundation.
Delete OpenShift Container Platform logging PVCs using OpenShift Data Foundation.
For more information, see, Removing the cluster logging operator from OpenShift Data Foundation.
Delete other PVCs and OBCs provisioned using OpenShift Data Foundation.
The following script is a sample script to identify the PVCs and OBCs provisioned using OpenShift Data Foundation. The script ignores the PVCs that are used internally by Openshift Data Foundation.

Raw
#!/bin/bash
RBD_PROVISIONER="openshift-storage.rbd.csi.ceph.com"
CEPHFS_PROVISIONER="openshift-storage.cephfs.csi.ceph.com"
NOOBAA_PROVISIONER="openshift-storage.noobaa.io/obc"
RGW_PROVISIONER="openshift-storage.ceph.rook.io/bucket"
NOOBAA_DB_PVC="noobaa-db"
NOOBAA_BACKINGSTORE_PVC="noobaa-default-backing-store-noobaa-pvc"
#Find all the OCS StorageClasses
OCS_STORAGECLASSES=$(oc get storageclasses | grep -e "$RBD_PROVISIONER" -e "$CEPHFS_PROVISIONER" -e         "$NOOBAA_PROVISIONER" -e "$RGW_PROVISIONER" | awk '{print $1}')
# List PVCs in each of the StorageClasses
for SC in $OCS_STORAGECLASSES
 do
   echo "======================================================================"
   echo "$SC StorageClass PVCs and OBCs"
   echo "======================================================================"
   oc get pvc  --all-namespaces --no-headers 2>/dev/null | grep $SC | grep -v -e "$NOOBAA_DB_PVC" -e "  $NOOBAA_BACKINGSTORE_PVC"
   oc get obc  --all-namespaces --no-headers 2>/dev/null | grep $SC
   echo
done
Note: Omit RGW_PROVISIONER for cloud platforms.

Delete the OBCs.

Raw
$ oc delete obc <obc name> -n <project name>
<obc-name> Is the name of the OBC
<project-name> Is the name of the project

Delete the PVCs.
Raw
$ oc delete pvc <pvc name> -n <project-name>
<pvc-name> Is the name of the PVC
<project-name> Is the name of the project

Note: Ensure that you have removed any custom backing stores, bucket classes, etc., created in the cluster.

Delete the Storage Cluster object and wait for the removal of the associated resources:

Raw
$ oc delete -n openshift-storage storagesystem --all --wait=true
IMPORTANT

If the resources are not deleted within 5 minutes, describe the object and determine the failure through the events or status:

Raw
$ oc describe -n openshift-storage <type> <name>
<type> Specify the type of the resource. It can be an API resource or object.
<name> Specify the name of the resource specified.

Example:

Raw
$ oc describe -n openshift-storage storagecluster ocs-storagecluster
Determine the required resolution for the failure and implement it.

Check for cleanup pods if the uninstall.ocs.openshift.io/cleanup-policy was set to delete(default) and ensure that their status is Completed.

Raw
$ oc get pods -n openshift-storage | grep -i cleanup
NAME                                READY   STATUS      RESTARTS   AGE
cluster-cleanup-job-<xx>         0/1     Completed   0          8m35s
cluster-cleanup-job-<yy>             0/1     Completed   0          8m35s
cluster-cleanup-job-<zz>             0/1     Completed   0          8m35s
Confirm that the directory /var/lib/rook is now empty. This directory will be empty only if the uninstall.ocs.openshift.io/cleanup-policy annotation was set to delete(default).

Raw
$ for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o jsonpath='{ .items[*].metadata.name  }'); do oc debug node/${i} -- chroot /host  ls -l /var/lib/rook; done
If encryption was enabled at the time of install, remove dm-crypt managed device-mapper mapping from OSD devices on all the OpenShift Data Foundation nodes.

Create a debug pod and chroot to the host on the storage node.

Raw
$ oc debug node/<node name>
$ chroot /host
<node-name> Is the name of the node

Get Device names and make note of the OpenShift Container Storage devices.

Raw
$ dmsetup ls
ocs-deviceset-0-data-0-57snx-block-dmcrypt (253:1)
Remove the mapped device.

Raw
$ cryptsetup luksClose --debug --verbose ocs-deviceset-0-data- 
0-57snx-block-dmcrypt
Note: If the above command gets stuck due to insufficient privileges, run the following commands:

Press CTRL+Z to exit the above command.
Find PID of the process which was stuck.

Raw
$ ps -ef | grep crypt
Terminate the process using the kill command.

Raw
$ kill -9 <PID>
<PID> Is the process ID

Verify that the device name is removed.

Raw
$ dmsetup ls
Delete the namespace and wait till the deletion is complete. You need to switch to another project if openshift-storage is the active project.

For example:

Raw
$ oc project default
$ oc delete project openshift-storage --wait=true --timeout=5m
The project is deleted if the following command returns a NotFound error.

Raw
$ oc get project openshift-storage
Note: While uninstalling OpenShift Data Foundation, if the namespace is not deleted completely and remains in Terminating state, perform the steps in Troubleshooting and deleting remaining resources during Uninstall to identify objects that are blocking the namespace from being terminated.

Note: Make sure to remove the disaster-protection finalizers in mon and mon-endpoints after this step. For more information, see Removing the disaster-protection finalizers in mon and mon endpoints.

Delete the local storage operator configurations if you have deployed OpenShift Data Foundation using local storage devices. See Removing local storage operator configurations.

Unlabel the storage nodes.

Raw
$ oc label nodes  --all cluster.ocs.openshift.io/openshift-storage-
$ oc label nodes  --all topology.rook.io/rack-
Remove the OpenShift Data Foundation taint if the nodes were tainted.

Raw
$ oc adm taint nodes --all node.ocs.openshift.io/storage-
Confirm all PVs provisioned using OpenShift Data Foundation are deleted. If there is any PV left in the Released state, delete it.

Raw
$ oc get pv
$ oc delete pv _<pv-name>_
<pv-name> Is the name of the PV

Delete the Multicloud Object Gateway storageclass.

Raw
$ oc delete storageclass openshift-storage.noobaa.io --wait=true --timeout=5m
Remove CustomResourceDefinitions.

Raw
oc delete crd backingstores.noobaa.io bucketclasses.noobaa.io cephblockpools.ceph.rook.io cephclusters.ceph.rook.io cephfilesystems.ceph.rook.io cephnfses.ceph.rook.io cephobjectstores.ceph.rook.io cephobjectstoreusers.ceph.rook.io noobaas.noobaa.io ocsinitializations.ocs.openshift.io storageclusters.ocs.openshift.io cephclients.ceph.rook.io cephobjectrealms.ceph.rook.io cephobjectzonegroups.ceph.rook.io cephobjectzones.ceph.rook.io cephrbdmirrors.ceph.rook.io storagesystems.odf.openshift.io cephblockpoolradosnamespaces.ceph.rook.io cephbucketnotifications.ceph.rook.io cephbuckettopics.ceph.rook.io cephcosidrivers.ceph.rook.io cephfilesystemmirrors.ceph.rook.io cephfilesystemsubvolumegroups.ceph.rook.io csiaddonsnodes.csiaddons.openshift.io networkfences.csiaddons.openshift.io reclaimspacecronjobs.csiaddons.openshift.io reclaimspacejobs.csiaddons.openshift.io storageclassrequests.ocs.openshift.io storageconsumers.ocs.openshift.io storageprofiles.ocs.openshift.io volumereplicationclasses.replication.storage.openshift.io volumereplications.replication.storage.openshift.io --wait=true --timeout=5m
(Optional) If cluster-wide encryption is enabled using the kubernetes authentication method for Key Management System (KMS), you need to remove the clusterrolebinding.

Raw
$ oc delete clusterrolebinding vault-tokenreview-binding
(Optional) To ensure that the vault keys are deleted permanently you need to manually delete the metadata associated with the vault key.

IMPORTANT
Execute this step only if Vault Key/Value (KV) secret engine API, version 2 is used for cluster-wide encryption with KMS since the vault keys are marked as deleted and not permanently deleted during the uninstallation of OpenShift Container Storage. You can always restore it later if required.

List the keys in the vault.

Raw
$ vault kv list <backend_path>
<backend_path> Is the path in the vault where the encryption keys are stored.
For example:

Raw
$ vault kv list kv-v2
Example output:

Raw
 Keys
 -----
NOOBAA_ROOT_SECRET_PATH/
rook-ceph-osd-encryption-key-ocs-deviceset-thin-0-data-0m27q8
rook-ceph-osd-encryption-key-ocs-deviceset-thin-1-data-0sq227
rook-ceph-osd-encryption-key-ocs-deviceset-thin-2-data-0xzszb
List the metadata associated with the vault key.
Raw
  $ vault kv get kv-v2/<key>
For the Multicloud Object Gateway (MCG) key:

Raw
  $ vault kv get kv-v2/NOOBAA_ROOT_SECRET_PATH/<key>
<key> Is the encryption key.

For Example:

Raw
$ vault kv get kv-v2/rook-ceph-osd-encryption-key-ocs-deviceset-thin-0-data-0m27q8
Example output:

Raw
====== Metadata ======
Key              Value
---              -----
created_time     2021-06-23T10:06:30.650103555Z
deletion_time    2021-06-23T11:46:35.045328495Z
destroyed        false
version          1
Delete the metadata.

Raw
$ vault kv metadata delete kv-v2/<key>
For the MCG key:

Raw
$ vault kv metadata delete kv-v2/NOOBAA_ROOT_SECRET_PATH/<key>
<key> Is the encryption key.
For Example:

Raw
$ vault kv metadata delete kv-v2/rook-ceph-osd-encryption-key-ocs-deviceset-thin-0-data-0m27q8
Example output:

Raw
Success! Data deleted (if it existed) at: kv-v2/metadata/rook-ceph-osd-encryption-key-ocs-deviceset-thin-0-data-0m27q8
Repeat these steps to delete the metadata associated with all the vault keys.
To ensure that OpenShift Data Foundation is uninstalled completely, on the OpenShift Container Platform Web Console.

Click Storage.
Verify that OpenShift Data Foundation no longer appears under Storage.
Removing local storage operator configurations
Use the instructions in this section only if you have deployed OpenShift Data Foundation using local storage devices.

Note: For OpenShift Data Foundation deployments only using localvolume resources, go directly to step 8.

Procedure

Identify the LocalVolumeSet and the corresponding StorageClassName being used by OpenShift Data Foundation.
Set the variable SC to the StorageClass providing the LocalVolumeSet.

Raw
 $ export SC="<StorageClassName>"
Delete the LocalVolumeSet.

Raw
 $ oc delete localvolumesets.local.storage.openshift.io <name-of-volumeset> -n openshift-local-storage
Delete the local storage PVs for the given StorageClassName.

Raw
$ oc get pv | grep $SC | awk '{print $1}'| xargs oc delete pv
Delete the StorageClassName.

Raw
$ oc delete sc $SC
Delete the symlinks created by the LocalVolumeSet.

Raw
$ [[ ! -z $SC ]] && for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o jsonpath='{ .items[*].metadata.name }'); do oc debug node/${i} -- chroot /host rm -rfv /mnt/local-storage/${SC}/; done
Delete the LocalVolumeDiscovery.

Raw
$ oc delete localvolumediscovery.local.storage.openshift.io/auto-discover-devices -n openshift-local-storage
Remove the LocalVolume resources (if any).

Use the following steps to remove the LocalVolume resources that were used to provision the PVs in the current or previous OpenShift Data Foundation version. Also, ensure that these resources are not being used by other tenants on the cluster.

For each of the local volumes, do the following:

Identify the LocalVolume and the corresponding StorageClassName being used by OpenShift Data Foundation.
Set the variable LV to the name of the LocalVolume and variable SC to the name of the StorageClass.
For example:

Raw
$ LV=localblock
$ SC=localblock
Delete the local volume resource.

Raw
$ oc delete localvolume -n openshift-local-storage --wait=true $LV
Delete the remaining PVs and StorageClasses if they exist.

Raw
$ oc delete pv -l storage.openshift.com/local-volume-owner-name=${LV} --wait --timeout=5m
$ oc delete storageclass $SC --wait --timeout=5m
Clean up the artifacts from the storage nodes for that resource.

Raw
$ [[ ! -z $SC ]] && for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o jsonpath='{ .items[*].metadata.name }'); do oc debug node/${i} -- chroot /host rm -rfv /mnt/local-storage/${SC}/; done
Example output:

Raw
Starting pod/node-xxx-debug ...
To use host binaries, run `chroot /host`
removed '/mnt/local-storage/localblock/nvme2n1'
removed directory '/mnt/local-storage/localblock'
Removing debug pod ...
Starting pod/node-yyy-debug ...
To use host binaries, run `chroot /host`
removed '/mnt/local-storage/localblock/nvme2n1'
removed directory '/mnt/local-storage/localblock'
Removing debug pod ...
Starting pod/node-zzz-debug ...
To use host binaries, run `chroot /host`
removed '/mnt/local-storage/localblock/nvme2n1'
removed directory '/mnt/local-storage/localblock'
Removing debug pod ...
Delete the openshift-local-storage namespace and wait till the deletion is complete. You will need to switch to another project if the openshift-local-storage namespace is the active project.

For Example:

Raw
$ oc project default
$ oc delete project openshift-local-storage --wait=true --timeout=5m
The project is deleted if the following command returns a NotFound error.

Raw
$ oc get project openshift-local-storage
Removing monitoring stack from OpenShift Data Foundation
Use this section to clean up the monitoring stack from OpenShift Data Foundation.

The Persistent Volume Claims (PVCs) that are created as a part of configuring the monitoring stack are in the openshift-monitoring namespace.

Prerequisites

PVCs are configured to use OpenShift Container Platform monitoring stack.
For more information, see configuring monitoring stack.

Procedure

List the pods and PVCs that are currently running in the openshift-monitoring namespace.

Raw
$ oc get pod,pvc -n openshift-monitoring
Example output:
NAME                           READY   STATUS    RESTARTS   AGE
pod/alertmanager-main-0         3/3     Running   0          8d
pod/alertmanager-main-1         3/3     Running   0          8d
pod/alertmanager-main-2         3/3     Running   0          8d
pod/cluster-monitoring-
operator-84457656d-pkrxm        1/1     Running   0          8d
pod/grafana-79ccf6689f-2ll28    2/2     Running   0          8d
pod/kube-state-metrics-
7d86fb966-rvd9w                 3/3     Running   0          8d
pod/node-exporter-25894         2/2     Running   0          8d
pod/node-exporter-4dsd7         2/2     Running   0          8d
pod/node-exporter-6p4zc         2/2     Running   0          8d
pod/node-exporter-jbjvg         2/2     Running   0          8d
pod/node-exporter-jj4t5         2/2     Running   0          6d18h
pod/node-exporter-k856s         2/2     Running   0          6d18h
pod/node-exporter-rf8gn         2/2     Running   0          8d
pod/node-exporter-rmb5m         2/2     Running   0          6d18h
pod/node-exporter-zj7kx         2/2     Running   0          8d
pod/openshift-state-metrics-
59dbd4f654-4clng                3/3     Running   0          8d
pod/prometheus-adapter-
5df5865596-k8dzn                1/1     Running   0          7d23h
pod/prometheus-adapter-
5df5865596-n2gj9                1/1     Running   0          7d23h
pod/prometheus-k8s-0            6/6     Running   1          8d
pod/prometheus-k8s-1            6/6     Running   1          8d
pod/prometheus-operator-
55cfb858c9-c4zd9                1/1     Running   0          6d21h
pod/telemeter-client-
78fc8fc97d-2rgfp                3/3     Running   0          8d

NAME                                                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
persistentvolumeclaim/my-alertmanager-claim-alertmanager-main-0   Bound    pvc-0d519c4f-15a5-11ea-baa0-026d231574aa   40Gi       RWO               ocs-storagecluster-ceph-rbd   8d
persistentvolumeclaim/my-alertmanager-claim-alertmanager-main-1   Bound    pvc-0d5a9825-15a5-11ea-baa0-026d231574aa   40Gi       RWO            ocs-storagecluster-ceph-rbd   8d
persistentvolumeclaim/my-alertmanager-claim-alertmanager-main-2   Bound    pvc-0d6413dc-15a5-11ea-baa0-026d231574aa   40Gi       RWO            ocs-storagecluster-ceph-rbd   8d
persistentvolumeclaim/my-prometheus-claim-prometheus-k8s-0        Bound    pvc-0b7c19b0-15a5-11ea-baa0-026d231574aa   40Gi       RWO            ocs-storagecluster-ceph-rbd   8d
persistentvolumeclaim/my-prometheus-claim-prometheus-k8s-1        Bound    pvc-0b8aed3f-15a5-11ea-baa0-026d231574aa   40Gi       RWO            ocs-storagecluster-ceph-rbd   8d
Edit the monitoring configmap.

Raw
$ oc -n openshift-monitoring edit configmap cluster-monitoring-config
Remove any config sections that reference the OpenShift Data Foundation storage classes as shown in the following example and save it.
Before editing:

Raw
[...]
apiVersion: v1
data:
 config.yaml: |
   alertmanagerMain:
     volumeClaimTemplate:
       metadata:
         name: my-alertmanager-claim
       spec:
         resources:
           requests:
             storage: 40Gi
         storageClassName: ocs-storagecluster-ceph-rbd
   prometheusK8s:
     volumeClaimTemplate:
       metadata:
         name: my-prometheus-claim
       spec:
         resources:
           requests:
             storage: 40Gi
         storageClassName: ocs-storagecluster-ceph-rbd
kind: ConfigMap
metadata:
 creationTimestamp: "2019-12-02T07:47:29Z"
 name: cluster-monitoring-config
 namespace: openshift-monitoring
 resourceVersion: "22110"
 selfLink: /api/v1/namespaces/openshift-monitoring/configmaps/cluster-monitoring-config
 uid: fd6d988b-14d7-11ea-84ff-066035b9efa8
[...].
After editing:

Raw
[...]
apiVersion: v1
data:
 config.yaml: |
kind: ConfigMap
metadata:
 creationTimestamp: "2019-11-21T13:07:05Z"
 name: cluster-monitoring-config
 namespace: openshift-monitoring
 resourceVersion: "404352"
 selfLink: /api/v1/namespaces/openshift-monitoring/configmaps/cluster-monitoring-config
 uid: d12c796a-0c5f-11ea-9832-063cd735b81c
[...]
In this example, alertmanagerMain and prometheusK8s monitoring components are using the OpenShift Data Foundation PVCs.

Delete the relevant PVCs. Make sure you delete all the PVCs that are consuming the storage classes.

Raw
$ oc delete -n openshift-monitoring pvc <pvc-name> --wait=true
<pvc-name> Is the name of the PVC

Removing OpenShift Container Platform registry from OpenShift Data Foundation
Use this section to clean up OpenShift Container Platform registry from OpenShift Data Foundation. If you want to configure alternative storage, see Image registry.

The Persistent Volume Claims (PVCs) that are created as a part of configuring the OpenShift Container Platform registry are in the openshift-image-registry namespace.

Prerequisites

The image registry should have been configured to use an OpenShift Data Foundation PVC.
Procedure

Edit the configs.imageregistry.operator.openshift.io object and remove the content in the storage section.

Raw
$ oc edit configs.imageregistry.operator.openshift.io
Before editing:

Raw
[...]
storage:
   pvc:
       claim: registry-cephfs-rwx-pvc
[...]
After editing:

Raw
[...]
storage:
  emptyDir: {}
[...]
In this example, the PVC is called registry-cephfs-rwx-pvc, which is now safe to delete.

Delete the PVC.

Raw
$ oc delete pvc <pvc-name> -n openshift-image-registry --wait=true
<pvc-name> Is the name of the PVC

Removing the cluster logging operator from OpenShift Data Foundation
Use this section to clean up the cluster logging operator from OpenShift Data Foundation.

The Persistent Volume Claims (PVCs) that are created as a part of configuring the cluster logging operator are in the openshift-logging namespace.

Prerequisites

The cluster logging instance should have been configured to use the OpenShift Data Foundation PVCs.
Procedure

Remove the ClusterLogging instance in the namespace.

Raw
$ oc delete clusterlogging instance -n openshift-logging --wait=true --timeout=5m
The PVCs in the openshift-logging namespace are now safe to delete.

Delete the PVCs.

Raw
$ oc delete pvc <pvc-name> -n openshift-logging --wait=true
<pvc-name> Is the name of the PVC

Removing RADOS Gateway (RGW) component
Use these instructions to disable only the RGW component.

Set ignore for RGW in the storage cluster.
Raw
$ oc patch -n openshift-storage storagecluster ocs-storagecluster \
    --type merge \
    --patch '{"spec": {"managedResources": {"cephObjectStores": {"reconcileStrategy": "ignore"}}}}'

$ oc patch -n openshift-storage storagecluster ocs-storagecluster \
    --type merge \                        
    --patch '{"spec": {"managedResources": {"cephObjectStoreUsers": {"reconcileStrategy": "ignore"}}}}'
Delete all RGW OBCs
Raw
$ oc get obc  --all-namespaces --no-headers |grep ceph 
#delete all obc from the list above
$ oc delete obc <obc> -n project
Delete RGW.
Raw
$ oc delete sc ocs-storagecluster-ceph-rgw
$ oc delete route s3-rgw
$ oc delete service rook-ceph-rgw-ocs-storagecluster-cephobjectstore
$ oc delete CephObjectStore ocs-storagecluster-cephobjectstore
Removing Multicloud Object Gateway component
Use these instructions to disable only the MCG component.

Set ignore for Multicloud Object Gateway by editing the storage cluster.
Raw
$ oc edit storagecluster 
Add the two lines marked with arrows.
Raw
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  multiCloudGateway:            #<---
    reconcileStrategy: ignore #<---
Delete all OBCs.
Raw
$ oc get obc  --all-namespaces --no-headers |grep noobaa 
#delete all OBCs from the list above
$ oc delete obc <obc> -n project
Delete NooBaa .
Raw
$ oc delete noobaa noobaa
Remove the default bucket as described below.
Removing the default bucket created by the Multicloud Object Gateway
The Multicloud Object Gateway (MCG) creates a default bucket in the cloud. You need to remove this default bucket.

NOTE: In case MCG is not deployed, you can skip this section.

Prerequisites

A running OpenShift Data Foundation Platform.
Download the MCG command-line interface for easier management:
Raw
# subscription-manager repos --enable=rh-odf-4-for-rhel-8-x86_64-rpms
Raw
# yum install mcg
IMPORTANT
Specify the appropriate architecture for enabling the repositories using the subscription manager.

For IBM Power, use the following command:
Raw
# subscription-manager repos --enable=rh-odf-4-for-rhel-8-ppc64le-rpms
For IBM Z infrastructure, use the following command:
Raw
# subscription-manager repos --enable=rh-odf-4-for-rhel-8-s390x-rpms
Alternatively, you can install the MCG package from the OpenShift Data Foundation RPMs found at the Download RedHat OpenShift Data Foundation page.
NOTE: Choose the correct Product Variant according to your architecture.

Procedure

Identify the target bucket that you need to remove:
Raw
$ noobaa backingstore status noobaa-default-backing-store
Example output:

Raw
[...]
# BackingStore spec:
s3Compatible:
    endpoint: https://**rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc:443**
    secret:
        name: rook-ceph-object-user-ocs-storagecluster-cephobjectstore-noobaa-ceph-objectstore-user
        namespace: openshift-storage
   signatureVersion: v4
   targetBucket: nb.1648032196977.apps.cluster1.vmware.ocp.team
type: s3-compatible

# Secret data:
[...]
In this example, rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc:443 is the endpoint which indicates the service that this backing store is using, for example, rook-ceph, Amazon Web Service (AWS) S3, Azure, and so on. The value in the targetBucket field is the name of the bucket that you need to delete.

Access the cloud or S3 compatible vendor, and delete the bucket identified in the previous step.
Removing the disaster-protection finalizers in mon and mon endpoints
Starting with 4.12, if you have configured disaster recovery for the clusters, you need to manually remove the 'ceph.rook.io/disaster-protection' finalizer from mon and the mon endpoints:

Edit configmap and secrets to remove the ceph.rook.io/disaster-protection finalizer from mon and mon endpoints.
Raw
$ oc edit configmap -n openshift-storage rook-ceph-mon-endpoints
$ oc edit secrets  -n openshift-storage rook-ceph-mon
Removing finalizer on the `ocs-client-operator-config` ConfigMap
If OpenShift Data Foundation version 4.16.0 or 4.16.1 is the initial deployment, remove the finalizer on the configmap as follows:

Raw
$ oc patch configmap ocs-client-operator-config -nopenshift-storage -p '{"metadata":{"finalizers":null}}' --type=merge
Article Type
