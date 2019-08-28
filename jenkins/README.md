# Jenkins Deployment

Jenkins deployment on LINSTOR storage:

```
$ kubectl apply -f jenkins-deployment.yaml
```

Check for socket on Kubernetes node:

```
$ kubectl get service jenkins
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
jenkins   NodePort   10.106.78.213   <none>        8080:31051/TCP   5s

$ curl -sI http://<node-name-or-ip>:$(kubectl get service jenkins | grep -o "8080:[0-9]*" | sed "s/8080://g") | grep ^HTTP
HTTP/1.1 200 OK
```

Open a browser and point it at the Jenkins service socket on any cluster node.
