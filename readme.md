# Hashicorp Vault Guide

Requirements:

* Kubernetes 1.24.2

For this tutorial, I will be using Kubernetes 1.24.2

Make sure you have Kubernetes Cluster Running, here I user 1 Master and 2 Worker

Clone this repo first
```
git clone
```

```
cd vault-2022
```

Install `helm`

```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

Now we have `helm` and `kubectl` and I can access my `kubernetes` cluster:

```
kubectl get nodes
NAME         STATUS   ROLES           AGE   VERSION
master       Ready    control-plane   15d   v1.24.2
worker1      Ready    <none>          15d   v1.24.2
worker2      Ready    <none>          15d   v1.24.2
```

Let's add the Helm repositories, so we can access the Kubernetes manifests

```
helm repo add hashicorp https://helm.releases.hashicorp.com
```

## Storage: Consul

We will use a very basic Consul cluster for our Vault backend. </br>
Let's find what versions of Consul are available:

```
helm search repo hashicorp/consul --versions
```

We can use chart `0.45.0` which is the latest at the time of this demo
Let's create a manifests folder and grab the YAML:

```

mkdir manifests

helm template consul hashicorp/consul \
  --namespace vault \
  --version 0.45.0 \
  -f consul-values.yaml \
  > ./manifests/consul.yaml
```

Make changes on consul.yaml file:
```
apiVersion: policy/v1beta1

to

apiVersion: policy/v1
 ```

Deploy the consul services:

```
kubectl create ns vault
kubectl -n vault create -f ./manifests/consul.yaml
kubectl -n vault get pods
```


## TLS End to End Encryption

See steps in [./tls/ssl_generate_self_signed.md](./tls/ssl_generate_self_signed.md)
You'll need to generate TLS certs (or bring your own)
Remember not to check-in your TLS to GIT :)

Create the TLS secret 

```
kubectl -n vault create secret tls tls-ca \
 --cert ./tls/ca.pem  \
 --key ./tls/ca-key.pem

kubectl -n vault create secret tls tls-server \
  --cert ./tls/vault.pem \
  --key ./tls/vault-key.pem
```

## Generate Kubernetes Manifests


Let's find what versions of vault are available:

```
helm search repo hashicorp/vault --versions
```

In this demo I will use the `0.20.1` chart </br>

Let's firstly create a `values` file to customize vault.
Let's grab the manifests:

```
helm template vault hashicorp/vault \
  --namespace vault \
  --version 0.20.1 \
  -f vault-values.yaml \
  > ./manifests/vault.yaml
```

Because I have only 2 workers, so I will change the vault.yaml for the vault replicas from 3 to 2

```
Before:
...
spec:
  serviceName: vault-internal
  podManagementPolicy: Parallel
  replicas: 3
  updateStrategy:
    type: OnDelete
...

after:
...
spec:
  serviceName: vault-internal
  podManagementPolicy: Parallel
  replicas: 2
  updateStrategy:
    type: OnDelete
...
```
## Deployment

```
kubectl -n vault create -f ./manifests/vault.yaml
kubectl -n vault get pods
```

## Initialising Vault

```
kubectl -n vault exec -it vault-0 -- vault operator init
```
Save the unseal key and the root token

## Uneeal Vault
```
kubectl -n vault exec -it vault-0 -- vault operator unseal
kubectl -n vault exec -it vault-1 -- vault operator unseal

kubectl -n vault exec -it vault-0 -- vault status
kubectl -n vault exec -it vault-1 -- vault status
```
## Web UI

Let's checkout the web UI:

```
kubectl -n vault get svc
kubectl -n vault port-forward svc/vault-ui 443:8200
```
Now we can access the web UI [here]("https://localhost/")

## Enable Kubernetes Authentication

For the injector to be authorised to access vault, we need to enable K8s auth

```
kubectl -n vault exec -it vault-0 -- sh 

vault login
vault auth enable kubernetes

vault write auth/kubernetes/config \
token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
issuer="https://kubernetes.default.svc.cluster.local"
exit
```

# Summary

So we have a vault, an injector, TLS end to end, stateful storage.
The injector can now inject secrets for pods from the vault.

Now we are ready to use the platform for different types of secrets:

## Secret Injection Guides

### Basic Secrets

Objective:
---------- 
* Let's create a basic secret in vault manually
* Application consumes the secret automatically

[Try it](./example-apps/basic-secret/readme.md)




