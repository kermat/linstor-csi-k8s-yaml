# LINSTOR Example YAML for Kubernetes

This repository has YAML definitions used to deploy demonstration applications onto [LINSTOR](https://github.com/LINBIT/linstor-csi) provisioned volumes in Kubernetes. These definitions are for demonstration purposes only, and do not follow best practices in terms of security for production use.

Since Kubernetes and LINSTOR can be configured in countless ways, these YAML definitions will assume that you have setup the [LINSTOR CSI plugin](https://github.com/LINBIT/linstor-csi) for Kubernetes, have a LINSTOR storage pool named `thin-lvm`, and have created a default storage class in Kubernetes named `csi-one-replica-autoplace-thin-lvm`. 

Use the following YAML (included in sc.yaml) to define the assumed storage class, or adjust it for your environment:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
  annotations:
  storageclass.kubernetes.io/is-default-class: "true"
metadata:
  name: "csi-one-replica-autoplace-thin-lvm"
parameters:
  autoplace: "1"
  storagePool: "thin-lvm"
provisioner: io.drbd.linstor-csi
reclaimPolicy: Delete
```

Use the `pvc.yaml` to test that you can create persistent volumes before trying the other examples in this repository:
```
$ kubectl apply -f pvc.yaml
persistentvolumeclaim/demo-vol-claim-0

$ kubectl get pvc
NAME                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                         AGE
demo-vol-claim-0                   Bound    pvc-4eff80d1-299e-11e9-adcd-a4badb334bb6   4Gi        RWO            csi-one-replica-autoplace-thin-lvm   7s
```
