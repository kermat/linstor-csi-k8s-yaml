# MySQL Replicated Stateful Set on LINSTOR Provisioned PV

This example was adapted from the [Kubernetes.io example](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/). Many of the sections of this README were copied verbatim.

The example is a MySQL single-master topology with multiple slaves running asynchronous replication.

Note that this is not a production configuration. In particular, MySQL settings remain on insecure defaults to keep the focus on general patterns for running stateful applications in Kubernetes on LINSTOR provisioned volumes.

## ConfigMap

Create the ConfigMap from the `mysql-configmap.yaml` YAML configuration file:
```
$ kubectl create -f mysql-configmap.yaml
```

This ConfigMap provides `my.cnf` overrides that let you independently control configuration on the MySQL master and slaves. In this case, you want the master to be able to serve replication logs to slaves and you want slaves to reject any writes that don’t come via replication.

There’s nothing special about the ConfigMap itself that causes different portions to apply to different Pods. Each Pod decides which portion to look at as it’s initializing, based on information provided by the StatefulSet controller.

## Services

Create the Services from the `mysql-services.yaml` YAML configuration file:
```
$ kubectl create -f mysql-services.yaml
```

The Headless Service provides a home for the DNS entries that the StatefulSet controller creates for each Pod that’s part of the set. Because the Headless Service is named `mysql`, the Pods are accessible by resolving `<pod-name>.mysql` from within any other Pod in the same Kubernetes cluster and namespace.

The Client Service, called `mysql-read`, is a normal Service with its own cluster IP that distributes connections across all MySQL Pods that report being Ready. The set of potential endpoints includes the MySQL master and all slaves.

Note that only read queries can use the load-balanced Client Service. Because there is only one MySQL master, clients should connect directly to the MySQL master Pod (through its DNS entry within the Headless Service) to execute writes.

## StatefulSet

Finally, create the StatefulSet from the mysql-statefulset.yaml YAML configuration file:
```
$ kubectl create -f mysql-statefulset.yaml
```

You can watch the startup progress by running: `kubectl get pods -l app=mysql --watch`

After a while, you should see all 3 Pods become Running:

```
NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          2m
mysql-1   2/2       Running   0          1m
mysql-2   2/2       Running   0          1m
```

Press **Ctrl+C** to cancel the watch. If you don’t see any progress, make sure you have the `linstor-csi-lvm-thin-r1` storage class defined, as mentioned in the prerequisites.

This manifest uses a variety of techniques for managing stateful Pods as part of a StatefulSet. The next section highlights some of these techniques to explain what happens as the StatefulSet creates Pods.

### Understanding stateful Pod initialization

The StatefulSet controller starts Pods one at a time, in order by their ordinal index. It waits until each Pod reports being Ready before starting the next one.

In addition, the controller assigns each Pod a unique, stable name of the form `<statefulset-name>-<ordinal-index>`. In this case, that results in Pods named `mysql-0`, `mysql-1`, and `mysql-2`.

The Pod template in the above StatefulSet manifest takes advantage of these properties to perform orderly startup of MySQL replication.

#### Generating configuration

Before starting any of the containers in the Pod spec, the Pod first runs any [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) in the order defined.

The first Init Container, named `init-mysql`, generates special MySQL config files based on the ordinal index.

The script determines its own ordinal index by extracting it from the end of the Pod name, which is returned by the `hostname` command. Then it saves the ordinal (with a numeric offset to avoid reserved values) into a file called `server-id.cnf` in the MySQL `conf.d` directory. This translates the unique, stable identity provided by the StatefulSet controller into the domain of MySQL server IDs, which require the same properties.

The script in the `init-mysql` container also applies either `master.cnf` or `slave.cnf` from the ConfigMap by copying the contents into `conf.d`. Because the example topology consists of a single MySQL master and any number of slaves, the script simply assigns ordinal `0` to be the master, and everyone else to be slaves. Combined with the StatefulSet controller’s [deployment order guarantee](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#deployment-and-scaling-guarantees/), this ensures the MySQL master is Ready before creating slaves, so they can begin replicating.

#### Cloning existing data

In general, when a new Pod joins the set as a slave, it must assume the MySQL master might already have data on it. It also must assume that the replication logs might not go all the way back to the beginning of time. These conservative assumptions are the key to allow a running StatefulSet to scale up and down over time, rather than being fixed at its initial size.

The second Init Container, named `clone-mysql`, performs a clone operation on a slave Pod the first time it starts up on an empty PersistentVolume. That means it copies all existing data from another running Pod, so its local state is consistent enough to begin replicating from the master.

MySQL itself does not provide a mechanism to do this, so the example uses a popular open-source tool called Percona XtraBackup. During the clone, the source MySQL server might suffer reduced performance. To minimize impact on the MySQL master, the script instructs each Pod to clone from the Pod whose ordinal index is one lower. This works because the StatefulSet controller always ensures Pod `N` is Ready before starting Pod `N+1`.

#### Starting replication

After the Init Containers complete successfully, the regular containers run. The MySQL Pods consist of a `mysql` container that runs the actual `mysqld` server, and an `xtrabackup` container that acts as a [sidecar](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns).

The `xtrabackup` sidecar looks at the cloned data files and determines if it’s necessary to initialize MySQL replication on the slave. If so, it waits for `mysqld` to be ready and then executes the `CHANGE MASTER TO` and `START SLAVE` commands with replication parameters extracted from the XtraBackup clone files.

Once a slave begins replication, it remembers its MySQL master and reconnects automatically if the server restarts or the connection dies. Also, because slaves look for the master at its stable DNS name (`mysql-0.mysql`), they automatically find the master even if it gets a new Pod IP due to being rescheduled.

Lastly, after starting replication, the `xtrabackup` container listens for connections from other Pods requesting a data clone. This server remains up indefinitely in case the StatefulSet scales up, or in case the next Pod loses its PersistentVolumeClaim and needs to redo the clone.

### Sending client traffic

You can send test queries to the MySQL master (hostname `mysql-0.mysql`) by running a temporary container with the `mysql:5.7` image and running the mysql client binary.
```
$ kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
```

Use the hostname `mysql-read` to send test queries to any server that reports being Ready:
```
$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-read -e "SELECT * FROM test.messages"
```

You should get output like this:
```
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

To demonstrate that the `mysql-read` Service distributes connections across servers, you can run `SELECT @@server_id` in a loop:
```
$ kubectl run mysql-client-loop --image=mysql:5.7 -i -t --rm --restart=Never --\
  bash -ic "while sleep 1; do mysql -h mysql-read -e 'SELECT @@server_id,NOW()'; done"
```

You should see the reported `@@server_id` change randomly, because a different endpoint might be selected upon each connection attempt:
```
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         100 | 2006-01-02 15:04:05 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         102 | 2006-01-02 15:04:06 |
+-------------+---------------------+
+-------------+---------------------+
| @@server_id | NOW()               |
+-------------+---------------------+
|         101 | 2006-01-02 15:04:07 |
+-------------+---------------------+
```

You can press **Ctrl+C** when you want to stop the loop, but it’s useful to keep it running in another window so you can see the effects of the following steps.

### Simulating Pod and Node downtime

To demonstrate the increased availability of reading from the pool of slaves instead of a single server, keep the SELECT @@server_id loop from above running while you force a Pod out of the Ready state.

#### Break the Readiness Probe

The readiness probe for the `mysql` container runs the command `mysql -h 127.0.0.1 -e 'SELECT 1'` to make sure the server is up and able to execute queries.

One way to force this readiness probe to fail is to break that command:
```
$ kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql /usr/bin/mysql.off
```

This reaches into the actual container’s filesystem for Pod `mysql-2` and renames the `mysql` command so the readiness probe can’t find it. After a few seconds, the Pod should report one of its containers as not Ready, which you can check by running: `kubectl get pod mysql-2`

Look for `1/2` in the `READY` column:
```
NAME      READY     STATUS    RESTARTS   AGE
mysql-2   1/2       Running   0          3m
```

At this point, you should see your `SELECT @@server_id` loop continue to run, although it never reports `102` anymore. Recall that the `init-mysql` script defined `server-id` as `100 + $ordinal`, so server ID `102` corresponds to Pod `mysql-2`.

Now repair the Pod and it should reappear in the loop output after a few seconds:
```
$ kubectl exec mysql-2 -c mysql -- mv /usr/bin/mysql.off /usr/bin/mysql
```

#### Delete Pods

The StatefulSet also recreates Pods if they’re deleted, similar to what a ReplicaSet does for stateless Pods.
```
$ kubectl delete pod mysql-2
```

The StatefulSet controller notices that no `mysql-2` Pod exists anymore, and creates a new one with the same name and linked to the same PersistentVolumeClaim. You should see server ID `102` disappear from the loop output for a while and then return on its own.

#### Drain a Node

If your Kubernetes cluster has multiple Nodes, you can simulate Node downtime (such as when Nodes are upgraded) by issuing a [drain](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#drain).

First determine which Node one of the MySQL Pods is on: `kubectl get pod mysql-2 -o wide`

The Node name should show up in the last column:
```
NAME      READY     STATUS    RESTARTS   AGE       IP            NODE
mysql-2   2/2       Running   0          15m       10.244.5.27   kubernetes-minion-group-9l2t
```

Then drain the Node by running the following command, which cordons it so no new Pods may schedule there, and then evicts any existing Pods. Replace `<node-name>` with the name of the Node you found in the last step.

This might impact other applications on the Node, so it’s best to **only do this in a test cluster**.
```
$ kubectl drain <node-name> --force --delete-local-data --ignore-daemonsets
```

Now you can watch as the Pod reschedules on a different Node: `kubectl get pod mysql-2 -o wide --watch`

It should look something like this:
```
NAME      READY   STATUS          RESTARTS   AGE       IP            NODE
mysql-2   2/2     Terminating     0          15m       10.244.1.56   kubernetes-minion-group-9l2t
[...]
mysql-2   0/2     Pending         0          0s        <none>        kubernetes-minion-group-fjlm
mysql-2   0/2     Init:0/2        0          0s        <none>        kubernetes-minion-group-fjlm
mysql-2   0/2     Init:1/2        0          20s       10.244.5.32   kubernetes-minion-group-fjlm
mysql-2   0/2     PodInitializing 0          21s       10.244.5.32   kubernetes-minion-group-fjlm
mysql-2   1/2     Running         0          22s       10.244.5.32   kubernetes-minion-group-fjlm
mysql-2   2/2     Running         0          30s       10.244.5.32   kubernetes-minion-group-fjlm
```

And again, you should see server ID `102` disappear from the `SELECT @@server_id` loop output for a while and then return.

Now uncordon the Node to return it to a normal state: `kubectl uncordon <node-name>`

### Scaling the number of slaves

With MySQL replication, you can scale your read query capacity by adding slaves. With StatefulSet, you can do this with a single command: `kubectl scale statefulset mysql  --replicas=5`

Watch the new Pods come up by running: `kubectl get pods -l app=mysql --watch`

Once they’re up, you should see server IDs `103` and `104` start appearing in the `SELECT @@server_id` loop output.

You can also verify that these new servers have the data you added before they existed:
```
$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

Scaling back down is also seamless: `kubectl scale statefulset mysql --replicas=3`

NOTE: while scaling up creates new PersistentVolumeClaims automatically, scaling down does not automatically delete these PVCs. This gives you the choice to keep those initialized PVCs around to make scaling back up quicker, or to extract data before deleting them.

You can see this by running: `kubectl get pvc -l app=mysql`

Which shows that all 5 PVCs still exist, despite having scaled the StatefulSet down to 3:
```
NAME           STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
data-mysql-0   Bound     pvc-8acbf5dc-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-1   Bound     pvc-8ad39820-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-2   Bound     pvc-8ad69a6d-b103-11e6-93fa-42010a800002   10Gi       RWO           20m
data-mysql-3   Bound     pvc-50043c45-b1c5-11e6-93fa-42010a800002   10Gi       RWO           2m
data-mysql-4   Bound     pvc-500a9957-b1c5-11e6-93fa-42010a800002   10Gi       RWO           2m
```

If you don’t intend to reuse the extra PVCs, you can delete them:
```
$ kubectl delete pvc data-mysql-3
$ kubectl delete pvc data-mysql-4
```

### Cleaning up

Cancel the `SELECT @@server_id` loop by pressing **Ctrl+C** in its terminal, or running the following from another terminal:
```
$ kubectl delete pod mysql-client-loop --now
```

Delete the StatefulSet. This also begins terminating the Pods.
```
$ kubectl delete statefulset mysql
```

Verify that the Pods disappear. They might take some time to finish terminating.
```
$ kubectl get pods -l app=mysql
```

You’ll know the Pods have terminated when the above returns: `No resources found.`

Delete the ConfigMap, Services, and PersistentVolumeClaims.
```
$ kubectl delete configmap,service,pvc -l app=mysql
```
