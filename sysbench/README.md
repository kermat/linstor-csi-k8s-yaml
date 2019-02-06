# Sysbench Examples for Generating IO

`sysbench` is a scriptable, open source, benchmarking suite. This directory includes example yamls for running benchmarks in Kubernetes, with the idea being to test performance of LINSTOR provisioned volumes used by some of the examples in this repo. 

## MySQL Benchmarking

If you followed the `mysql-replicated-stateful-set` example in this repo, you'd have a `mysql-0.mysql` pod acting as Master, with some number of read-only Slave pods, all running on top of LINSTOR provisioned volumes in you Kubernetes cluster.

To test the performance of that MySQL stateful set first create a test database on `mysql-0.mysql`:
```
$ kubectl exec -it mysql-0 -- /usr/bin/mysql -u root -e 'create database sbtest;'
```

Then, run the job file in this directory:
```
$ kubectl apply -f sysbench-job.yaml
```

If you describe the job you will see that it's running, but not yet completed (Succeeded):
```
$ kubectl describe jobs sysbench-oltp
```

Watch for the job to complete:
```
$ kubectl get jobs -w
```

Once completed, you can review the results by viewing the standard out of the job's pod:
```
$ pods=$(kubectl get pods --selector=job-name=sysbench-oltp --output=jsonpath={.items..metadata.name})
$ kubectl logs $pods
```

Finally, delete the job:
```
$ kubectl delete jobs.batch sysbench-oltp
```
