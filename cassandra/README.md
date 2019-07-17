# Cassandra on LINSTOR

```
$ kubectl apply -f cassandra-service.yaml
service "cassandra" configured

$ kubectl apply -f cassandra-statefulset.yaml
statefulset "cassandra" created
```

## Verify Cassandra Health

```
$ kubectl exec cassandra-0 -- nodetool status
Datacenter: DC1-K8Demo
======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address      Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.244.0.20  141.31 MiB  32           32.9%             3159df0b-9bd9-49d8-a7d7-2bee9ac00409  Rack1-K8Demo
UN  10.244.1.35  119.97 MiB  32           30.8%             4bd3574c-a1f7-4dfd-9cff-3e5612191665  Rack1-K8Demo
UN  10.244.2.40  130.12 MiB  32           36.4%             74604286-3112-43a8-b288-7a8ce1a258a2  Rack1-K8Demo
```

NOTE: "UN" stands for Up and Normal

## Test Cassandra Performance on LINSTOR

Use `cassandra-loadgen.yaml` to test the performance of Cassandra on LINSTOR.

Run the Cassandra loadgen Kubernetes job:

```
$ kubectl apply -f cassandra-loadgen.yaml
job "cassandra-loadgen" created
```

View the results and watch the progress by inspecting or following the logs for the job:

```
$ kubectl logs -f cassandra-loadgen
******************** Stress Settings ********************
Command:
  Type: write
  Count: -1
  Duration: 5 MINUTES
  No Warmup: true
  Consistency Level: LOCAL_ONE
  Target Uncertainty: not applicable
  Key Size (bytes): 10
  Counter Increment Distribution: add=fixed(1)
<snip>
Running with 121 threadCount
Running WRITE with 121 threads 5 minutes
Failed to connect over JMX; not collecting these stats
type       total ops,    op/s,    pk/s,   row/s,    mean,     med, <snip>
total,           280,     280,     280,     280,    28.8,    32.6, <snip>
total,          2413,    2133,    2133,    2133,    55.8,    90.5, <snip>
total,          4309,    1896,    1896,    1896,    63.8,    95.4, <snip>
total,          6191,    1882,    1882,    1882,    65.2,    94.4, <snip>
total,          8088,    1897,    1897,    1897,    64.2,    87.8, <snip>
```
