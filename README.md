# Let's Play with Kubernetes

## Create a Kubernetes Cluster

### Create a Kubernetes Cluster with Azure Kubernetes Service (AKS)

You can easily create a Kubernetes cluster on Azure with the `az` CLI just by typing a few commands.

Let's first capture within environment variables the Azure _resource group_ name and the Kubernetes _cluster_ name so you can have a custom installation while just copy-pasting the commands below.

```text
$ group="my-resource-group"
$ echo $group
my-resource-group
$ cluster="my-cluster"
$ echo $cluster
my-cluster
```

```text
$ az group create --name $group --location westeurope
...

$ az aks create \
  --resource-group $group \
  --name $cluster \
  --enable-managed-identity \
  --generate-ssh-keys
...

$ az aks get-credentials \
  --resource-group $group \
  --name $cluster
Merged "my-cluster" as current context in /home/user/.kube/config

$ kubectl version --short
Client Version: v1.21.1
Server Version: v1.19.11
WARNING: version difference between client (1.21) and server (1.19) exceeds the supported minor version skew of +/-1
```

## Scenario 1

### Description of the Scenario 1

This scenario illustrates an HTTP client service calling one HTTP API endpoint every second.

The API endpoint will be `GET /uuid` implemented by the `kennethreitz/httpbin` image.

The HTTP client service will be a script using `curl`.

We will call `service-b` the HTTP server service implementing the `GET /uuid` API endpoint.

We will call `service-a` the HTTP client service.

#### Configuration of the `service-b`

The configuration of the `service-b` will be contained within a dedicated `service-b-ns` Namespace.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: service-b-ns
```

</details>

The HTTP server service `service-b` will use `service-b-sa` ServiceAccount as identity.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-b-sa
  namespace: service-b-ns
```

</details>

The HTTP server service `service-b` will be deployed within the Kubernetes cluster with a `service-b-deploy` Deployment running one pod containing one container `service-b-co`, using  the `kennethreitz/httpbin` image, with the `service-b-sa` identity.

<details>
<summary>YAML</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b-deploy
  namespace: service-b-ns
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

</details>

The HTTP server service `service-b` will be exposed within the Kubernetes cluster with the hostname `service-b.service-b-ns` on port `80`.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-b
  namespace: service-b-ns
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

</details>

#### Configuration of the `service-a`

The configuration of the `service-a` will contain within a dedicated `service-a-ns` Namespace.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: service-a-ns
```

</details>


The HTTP client service `service-a` will use `service-a-sa` ServiceAccount as identity

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-a-sa
  namespace: service-a-ns
```

</details>

The HTTP client service `service-a` will be implemented will the following Bash script using `curl`

```text
while curl http://service-b.service-b-ns/uuid; do sleep 1.0; done;
```

The HTTP client service `service-a` will be deployed within the Kubernetes cluster with a `service-a-deploy` Deployment running one pod containing one container `service-a-co`, using  the `tutum/curl` image and the Bash script above, with the `service-a-sa` identity.

<details>
<summary>YAML</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a-deploy
  namespace: service-a-ns
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
        args: ["-c", "while curl http://service-b.service-b-ns/uuid; do sleep 1.0; done;"]
```

</details>

### Running Scenario 1

You can install all the components of the scenario 1 using the `service-b.yaml` and `service-a.yaml` files using the following commands:

```text
$ cd scenario-01/

$ kubectl apply -f service-b.yaml
namespace/s01-service-b-ns created
serviceaccount/service-b-sa created
deployment.apps/service-b-deploy created
service/service-b created

$ kubectl apply -f service-a.yaml
namespace/s01-service-a-ns created
serviceaccount/service-a-sa created
deployment.apps/service-a-deploy created
```

```text
$ kubectl get pods -A
NAME                                READY   STATUS    RESTARTS   AGE
...
s01-service-a-ns     service-a-deploy-d6777698-7kg4j            1/1     Running   0          21s
s01-service-b-ns     service-b-deploy-689459595d-cz8mp          1/1     Running   0          51s
...
```

```text
$ kubectl logs -n s01-service-a-ns --tail 20 service-a-deploy-5c6c5df758-4v2s6
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
