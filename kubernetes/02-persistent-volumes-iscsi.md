# [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

A **PersistentVolume (PV)** is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. A **PersistentVolumeClaim (PVC)** is a request for storage by a user. It is similar to a Pod.

A PV can have a **class**, which is specified by setting the storageClassName attribute to the name of a StorageClass. A PV of a particular class can only be bound to PVCs requesting that class.

## Prerequisites (Preparing iSCSI Target and Initiator)

See [iSCSI server](../storage/iscsi/rhel-target.md) and [iSCSI client](../storage/iscsi/rhel-initiator.md) for more instructions.

**iqn** and **iSCSI server ip** are required to finish this section.

## Create a storage class

```bash
kubectl create -f iscsi-storage-class.yaml
```

## Add authenthiation credential

```bash
kubectl create -f iscsi-chap-secret.yaml
```

## Add PersistentVolume (PV) luns

```bash
kubectl create -f iscsi-chap-pv-lun0.yaml

kubectl get pv

# Output should be similar to following:
[onme@unimog ~]$ kubectl get pv
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                  STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
iscsi-pv0   64Gi       RWO            Retain           Bound       billions/grafana-pvc   manual         <unset>                          9d
```

You can add more PVs as long as you have enough luns on your iSCSI server by editing *iscsi-chap-pv-lun0.yaml*

## Test a PersistentVolumeClaim (PVC)

* Create a local pvc.yaml file with following content:

```bash
# cat pvc.yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zzz-iscci-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
  storageClassName: zzz-iscsi

```

* Provision the persistent volume

```bash
kubectl apply -f pvc.yaml

# Output should be similar to following:
persistentvolumeclaim/zzz-iscci-pvc created

kubectl get pvc

# Output should be similar to following:
[onme@unimog Tsst]$ kubectl get pvc
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
zzz-iscci-pvc   Pending                                      manual         <unset>                 3m31s

kubectl describe pvc

# Output should be similar to following:
[onme@unimog Tsst]$ kubectl describe pvc
Name:          zzz-iscci-pvc
Namespace:     default
StorageClass:  manual
Status:        Pending
Volume:        
Labels:        <none>
Annotations:   <none>
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      
Access Modes:  
VolumeMode:    Filesystem
Used By:       <none>
Events:
  Type    Reason                Age                   From                         Message
  ----    ------                ----                  ----                         -------
  Normal  WaitForFirstConsumer  11s (x12 over 2m50s)  persistentvolume-controller  waiting for first consumer to be created before binding

# Clean up
kubectl delete -f pvc.yaml
kubectl describe pvc

# Output should be similar to following:
[onme@unimog Tsst]$ kubectl describe pvc
No resources found in default namespace.

```
