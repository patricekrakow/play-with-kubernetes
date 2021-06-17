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

We will contain this scenario within a dedicated `scenario-01-ns` Namespace.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: scenario-01-ns
```

</details>

The HTTP server service `service-b` will use `service-b-sa` ServiceAccount as identity.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: scenario-01-ns
  name: service-b-sa
```

</details>

The HTTP client service `service-a` will use `service-a-sa` ServiceAccount as identity

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: scenario-01-ns
  name: service-a-sa
```

</details>

The HTTP server service `service-b` will be deployed within the Kubernetes cluster with a `service-b-deploy` Deployment running one pod containing one container `service-b-co`, using  the `kennethreitz/httpbin` image, with the `service-b-sa` identity.

<details>
<summary>YAML</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: scenario-01-ns
  name: service-b-deploy
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

The HTTP server service `service-b` will be exposed within the Kubernetes cluster with the hostname `service-b.scenario-01-ns` on port `8080`.

<details>
<summary>YAML</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: scenario-01-ns
  name: service-b
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

The HTTP client service `service-a` will be implemented will the following Bash script using `curl`

```text
while curl http://service-b.scenario-01-ns/uuid; do sleep 1.0; done;
```

The HTTP client service `service-a` will be deployed within the Kubernetes cluster with a `service-a-deploy` Deployment running one pod containing one container `service-a-co`, using  the `tutum/curl` image and the Bash script above, with the `service-a-sa` identity.

<details>
<summary>YAML</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: scenario-01-ns
  name: service-a-deploy
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
        args: ["-c", "while curl http://service-b.scenario-01-ns/uuid; do sleep 1.0; done;"]
```

</details>

### Running Scenario 1

You can install all the components of the scenario 1 using the `scenario-01.yaml` file using the following command:

```text
$ kubectl apply -f scenario-01.yaml
namespace/scenario-01-ns created
serviceaccount/service-b-sa created
serviceaccount/service-a-sa created
deployment.apps/service-b-deploy created
service/service-b created
deployment.apps/service-a-deploy created
```

```text
$ kubectl get pods -n scenario-01-ns
NAME                                READY   STATUS    RESTARTS   AGE
service-a-deploy-5c6c5df758-4v2s6   1/1     Running   0          53s
service-b-deploy-689459595d-42vgz   1/1     Running   0          53s
```

```text
$ k logs service-a-deploy-5c6c5df758-4v2s6 -n scenario-01-ns --tail 20
  "uuid": "1852095d-d11b-4b34-b75b-03221ac3b48c"
}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0   8277      0 --:--:-- --:--:-- --:--:--  8833
{
  "uuid": "b274147e-f9e5-4a0a-8b17-449236869197"
}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0   7991      0 --:--:-- --:--:-- --:--:--  8833
{
  "uuid": "f1ab5bd8-1359-4ebc-ab17-3be1494c1c67"
}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
{
  "uuid": "af910b32-26ca-43d1-b95d-2ac7c940932c"
}
100    53  100    53    0     0   7843      0 --:--:-- --:--:-- --:--:--  8833
```
