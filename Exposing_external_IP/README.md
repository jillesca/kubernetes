# Exposing an External IP Address to Access an Application in a Cluster

This is an exercise from [kubernetes.io](https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/)

## Objectives

- Run five instances of a Hello World application.
- Create a Service object that exposes an external IP address.
- Use the Service object to access the running application.

## Steps

Since I'm testing on my laptop

```bash
minikube start
```

Start the deployment

```
╰─ kubectl apply -f load-balancer-example.yaml                                                    ─╯
deployment.apps/hello-world created
```

Display information about the deployment

```
╰─ kubectl get deployments.apps hello-world                                                       ─╯
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
hello-world   0/5     5            0           36s

╰─ kubectl describe deployments.apps hello-world                                                                                       ─╯
Name:                   hello-world
Namespace:              default
CreationTimestamp:      Wed, 28 Sep 2022 07:54:36 +0100
Labels:                 app.kubernetes.io/name=load-balancer-example
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app.kubernetes.io/name=load-balancer-example
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
...
```

## See ReplicaSet objects

```bash
╰─ kubectl get replicasets.apps                                                                                                        ─╯
NAME                     DESIRED   CURRENT   READY   AGE
hello-world-75f87549d8   5         5         5       6m8s

╰─ kubectl describe replicasets.apps                                                                                                   ─╯
Name:           hello-world-75f87549d8
Namespace:      default
Selector:       app.kubernetes.io/name=load-balancer-example,pod-template-hash=75f87549d8
Labels:         app.kubernetes.io/name=load-balancer-example
                pod-template-hash=75f87549d8
Annotations:    deployment.kubernetes.io/desired-replicas: 5
                deployment.kubernetes.io/max-replicas: 7
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/hello-world
Replicas:       5 current / 5 desired
Pods Status:    5 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

## Create a service object that exposes the deployment

If a this point I try to see the hello world containers, it will fail.

```bash
╰─ kubectl get services                                                                                                                ─╯
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   9m26s

╰─ ping 10.96.0.1 -c 4                                                                                                                 ─╯
PING 10.96.0.1 (10.96.0.1): 56 data bytes
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
Request timeout for icmp_seq 2

--- 10.96.0.1 ping statistics ---
4 packets transmitted, 0 packets received, 100.0% packet loss

```bash
╰─ curl 10.96.0.1 -vvv                                                                                                                 ─╯
*   Trying 10.96.0.1:80...
* connect to 10.96.0.1 port 80 failed: Operation timed out
* Failed to connect to 10.96.0.1 port 80 after 75020 ms: Operation timed out
* Closing connection 0
curl: (28) Failed to connect to 10.96.0.1 port 80 after 75020 ms: Operation timed out
```

The reason is that we need to create a service to expose the POD.

> The k8s guide uses a `LoadBalancer` type of service, but when using minikube `NodePort` is only available.

```bash
╰─ kubectl expose deployment hello-world --type=NodePort --port=8080 --name=my-service                                                 ─╯
service/my-service exposed

╰─ kubectl get services my-service                                                                                                     ─╯
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
my-service   NodePort   10.98.157.104   <none>        8080:32375/TCP   15s

╰─ kubectl describe services my-service                                                                                         ─╯
Name:                     my-service
Namespace:                default
Labels:                   app.kubernetes.io/name=load-balancer-example
Annotations:              <none>
Selector:                 app.kubernetes.io/name=load-balancer-example
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.98.157.104
IPs:                      10.98.157.104
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32375/TCP
Endpoints:                172.17.0.2:8080,172.17.0.3:8080,172.17.0.4:8080 + 2 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Finally, on minikube you need to open a tunnel to reach your containers

```bash
╰─ minikube service list                                                                                                               ─╯
|-------------|------------|--------------|-----|
|  NAMESPACE  |    NAME    | TARGET PORT  | URL |
|-------------|------------|--------------|-----|
| default     | kubernetes | No node port |
| default     | my-service |         8080 |     |
| kube-system | kube-dns   | No node port |
|-------------|------------|--------------|-----|


╰─ minikube service my-service                                                                                                         ─╯
|-----------|------------|-------------|---------------------------|
| NAMESPACE |    NAME    | TARGET PORT |            URL            |
|-----------|------------|-------------|---------------------------|
| default   | my-service |        8080 | http://192.168.49.2:32375 |
|-----------|------------|-------------|---------------------------|
🏃  Starting tunnel for service my-service.
|-----------|------------|-------------|------------------------|
| NAMESPACE |    NAME    | TARGET PORT |          URL           |
|-----------|------------|-------------|------------------------|
| default   | my-service |             | http://127.0.0.1:61401 |
|-----------|------------|-------------|------------------------|
🎉  Opening service default/my-service in default browser...
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

After this, in your browser or another terminal, you will be able to reach your containers.

```bash
╰─ curl 127.0.0.1:61401 -vvv                                                                                                    ─╯
*   Trying 127.0.0.1:61401...
* Connected to 127.0.0.1 (127.0.0.1) port 61401 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:61401
> User-Agent: curl/7.79.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Thu, 29 Sep 2022 08:20:53 GMT
< Connection: keep-alive
< Transfer-Encoding: chunked
<
* Connection #0 to host 127.0.0.1 left intact
Hello Kubernetes!%
```
