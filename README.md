# Play with Kubernetes

## Create a Kubernetes Cluster

### Create a Kubernetes Cluster with Azure Kubernetes Service (AKS)

You can easily create a Kubernetes cluster on Azure with the `az` CLI by typing the following commands:

```text
$ az group create --name pk-group-01 --location westeurope
...

$ az aks create \
  --resource-group pk-group-01 \
  --name pk-cluster-01 \
  --enable-managed-identity \
  --generate-ssh-keys
...

$ az aks get-credentials \
  --resource-group pk-group-01 \
  --name pk-cluster-01
Merged "pk-cluster-01" as current context in /home/patrice_krakow/.kube/config

$ kubectl version --short
Client Version: v1.20.4
Server Version: v1.18.14
```

## Description of scenario #1

This scenario illustrates an HTTP client service calling one HTTP API endpoint every second.

The HTTP client service will be a script using `curl`.

The API endpoint will be `GET /uuid` implemented by the `kennethreitz/httpbin` container.

We will call `service-a` the HTTP client service.

We will call `service-b` the HTTP server service implementing the `GET /uuid` API endpoint.

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
