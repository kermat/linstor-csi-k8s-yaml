# RabbitMQ on LINSTOR Provisioned PV

The following should be all you need to test the deployment of a RabbitMQ stateful set onto LINSTOR provisioned volumes in Kubernetes.

Create erlang cookie/secret:
```
$ kubectl create secret generic rabbitmq-config --from-literal=erlang-cookie=rabbitmq-linstor-123
```

Create the StatefulSet:
```
$ kubectl apply -f rabbitmq-statefulset.yaml
```

## Cleanup
```
$ kubectl delete -f rabbitmq-statefulset.yaml
$ kubectl delete secrets rabbitmq-config
$ kubectl delete pvc -l app=rabbitmq
```
