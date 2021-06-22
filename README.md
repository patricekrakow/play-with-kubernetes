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
namespace/service-b-ns created
serviceaccount/service-b-sa created
deployment.apps/service-b-deploy created
service/service-b created

$ kubectl apply -f service-a.yaml
namespace/service-a-ns created
serviceaccount/service-a-sa created
deployment.apps/service-a-deploy created
```

```text
$ kubectl get pods -A
NAME                                READY   STATUS    RESTARTS   AGE
...
service-a-ns     service-a-deploy-d6777698-nxthc            1/1     Running   0          2m19s
service-b-ns     service-b-deploy-689459595d-w9rn9          1/1     Running   0          3m9s
...
```

```text
$ kubectl logs -n scenario-01-ns --tail 20 service-a-deploy-5c6c5df758-4v2s6
}
100    53  100    53    0     0  11255      0 --:--:-- --:--:-- --:--:-- 13250
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "0a87d0a9-d878-491b-bbdb-fe520b612add"
}
100    53  100    53    0     0  12257      0 --:--:-- --:--:-- --:--:-- 13250
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "09cbd036-bfa9-4d53-85e3-b135e949f41c"
}
100    53  100    53    0     0   9069      0 --:--:-- --:--:-- --:--:-- 10600
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "1dc727cc-d1df-4dd2-9b34-6065b4230b71"
}
100    53  100    53    0     0  11240      0 --:--:-- --:--:-- --:--:-- 13250
```
