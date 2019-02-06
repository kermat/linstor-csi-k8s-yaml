# Redis Cluster on LINSTOR Provisioned PV

This document demonstrates the deployment of Redis as a StatefulSet in a Kubernetes cluster. The user can spawn a Redis StatefulSet that will use LINSTOR as its persistent storage.

Before getting started check the status of the cluster:

```bash
$ kubectl get nodes
NAME            STATUS    AGE       VERSION
gimli.us.linbit    Ready     <none>    17d       v1.13.2
merry.us.linbit    Ready     <none>    17d       v1.13.2
pippin.us.linbit   Ready     master    17d       v1.13.2
```

Apply the `reddis-statefulset.yaml` YAML from this repository:

```bash
$ kubectl apply -f redis-statefulset.yaml
```

Get the status of running pods, after the init process finishes, you should see the following:

```bash
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
rd-0      1/1     Running   0          8m6s
rd-1      1/1     Running   0          6m44s
rd-2      1/1     Running   0          5m28s
```

Get the status of running StatefulSets:

```bash
$ kubectl get statefulset
NAME      DESIRED   CURRENT   AGE
rd        3         3         19h
```

Get the status of underlying persistent volumes used by Redis StatefulSet:

```bash
$ kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                         AGE
datadir-rd-0   Bound    pvc-d180347e-2658-11e9-8b47-a4badb334bb6   954Mi      RWO            csi-one-replica-autoplace-thin-lvm   7m21s
datadir-rd-1   Bound    pvc-024f7008-2659-11e9-8b47-a4badb334bb6   954Mi      RWO            csi-one-replica-autoplace-thin-lvm   5m59s
datadir-rd-2   Bound    pvc-2f69bb1c-2659-11e9-8b47-a4badb334bb6   954Mi      RWO            csi-one-replica-autoplace-thin-lvm   4m43s
```

Get the status of the services:

```bash
$ kubectl get svc
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    17d
redis         ClusterIP   None            <none>        6379/TCP   9m13s
```

## Check Redis Replication

Set a key:value pair in the Redis master.

```bash
$ kubectl exec rd-0 -- /opt/redis/redis-cli -h rd-0.redis SET replicated:test true
OK
```

Retrieve the value of the key from a Redis slave.

```bash
$ kubectl exec rd-2 -- /opt/redis/redis-cli -h rd-0.redis GET replicated:test
true
```
