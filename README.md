# Let's Play with Kubernetes

As usual, let's go to <https://killercoda.com/playgrounds/scenario/kubernetes> to get a Kubernetes cluster (for 60 minutes).

## Scenario 1

### Description of the Scenario 1

***NOTE:*** If you want, you can jump to the [Running Scenario 1](#running-scenario-1) section to directly see things running!

This scenario illustrates an HTTP client service calling one HTTP API endpoint every second.

The API endpoint will be `GET /uuid` implemented by the `kennethreitz/httpbin` image.

The HTTP client service will be a script using `curl`.

We will call `service-b` the HTTP server service implementing the `GET /uuid` API endpoint.

We will call `service-a` the HTTP client service.

![Diagram 1](play-with-kubernetes.scenario-01.jpg)

_Fig. 1: Scenario 1_

#### Configuration of the `service-b`

The configuration of the `service-b` will be contained within a dedicated `s01-service-b-ns` Namespace. You can noticed that we prefix the namespace `s01-` which refers to "scenario 1". We did that to run different scenarios at the same time on the same cluster, without having interferences between them.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: s01-service-b-ns
```

The HTTP server service `service-b` will use `service-b-sa` ServiceAccount as identity.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-b-sa
  namespace: s01-service-b-ns
```

The HTTP server service `service-b` will be deployed within the Kubernetes cluster with a `service-b-deploy` Deployment running one pod containing one container `service-b-co`, using  the `kennethreitz/httpbin` image, with the `service-b-sa` identity.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b-deploy
  namespace: s01-service-b-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-b
      version: 0.0.1
  template:
    metadata:
      labels:
        app: service-b
        version: 0.0.1
    spec:
      serviceAccountName: service-b-sa
      containers:
      - name: service-b-co
        image: kennethreitz/httpbin
        ports:
        - containerPort: 80
```

The HTTP server service `service-b` will be exposed within the Kubernetes cluster with the hostname `service-b.s01-service-b-ns` on port `80`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-b
  namespace: s01-service-b-ns
spec:
  selector:
    app: service-b
    version: 0.0.1
  ports:
  - name: service-b-http-port
    protocol: TCP
    port: 80
    targetPort: 80
```

#### Configuration of the `service-a`

The configuration of the `service-a` will contain within a dedicated `s01-service-a-ns` Namespace.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: s01-service-a-ns
```

The HTTP client service `service-a` will use `service-a-sa` ServiceAccount as identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-a-sa
  namespace: s01-service-a-ns
```

The HTTP client service `service-a` will be implemented will the following Bash script using `curl`

```text
while curl http://service-b.s01-service-b-ns/uuid; do sleep 1.0; done;
```

The HTTP client service `service-a` will be deployed within the Kubernetes cluster with a `service-a-deploy` Deployment running one pod containing one container `service-a-co`, using  the `tutum/curl` image and the Bash script above, with the `service-a-sa` identity.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a-deploy
  namespace: s01-service-a-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: service-a
      version: 0.0.1
  template:
    metadata:
      labels:
        app: service-a
        version: 0.0.1
    spec:
      serviceAccountName: service-a-sa
      containers:
      - name: service-a-co
        image: curlimages/curl
        command: ["/bin/sh"]
        args: ["-c", "while curl http://service-b.s01-service-b-ns/uuid; do sleep 1.0; done;"]
```

### Running Scenario 1

Let's first download the manifest files:

```text
git clone https://github.com/patricekrakow/play-with-kubernetes.git
```

```text
cd play-with-kubernetes/scenario-01/
```

You can install all the components of the scenario 1 using the `service-b.yaml` and `service-a.yaml` files using the following commands:

```text
kubectl apply -f service-b.yaml
```
```text
namespace/s01-service-b-ns created
serviceaccount/service-b-sa created
deployment.apps/service-b-deploy created
service/service-b created
```

```text
kubectl apply -f service-a.yaml
```
```text
namespace/s01-service-a-ns created
serviceaccount/service-a-sa created
deployment.apps/service-a-deploy created
```

You can now verify the installation:

```text
kubectl get pods -A
```
```text
NAME                                READY   STATUS    RESTARTS   AGE
...
s01-service-a-ns     service-a-deploy-d6777698-7kg4j            1/1     Running   0          21s
s01-service-b-ns     service-b-deploy-689459595d-cz8mp          1/1     Running   0          51s
...
```

And, finally, check that the `service-a` is calling the `GET /uuid` API endpoint implemented by the `service-b`:

```text
kubectl logs -n s01-service-a-ns --tail 20 service-a-deploy-5c6c5df758-4v2s6
```
```text
}
100    53  100    53    0     0  10329      0 --:--:-- --:--:-- --:--:-- 10600
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0  10245      0 --:--:-- --:--:-- --:--:-- 10600
{
  "uuid": "3d56ddf1-411c-4050-a803-c2380c8b5d93"
}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "78cae782-2ceb-489c-8373-8cb38aff1bd1"
}
100    53  100    53    0     0  11039      0 --:--:-- --:--:-- --:--:-- 13250
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "8e23020d-6765-400a-874e-d7f8b3574c0d"
}
100    53  100    53    0     0  10998      0 --:--:-- --:--:-- --:--:-- 13250
```
