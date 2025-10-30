# Azure AKS - ArgoCD Tutorial

In this tutorial we are going to setup a Azure AKS Cluster and deploy ArgoCD and a Example App in it.
This tutorial is based on the [ArgoCD for Beginners Tutorial](https://www.youtube.com/watch?v=MeU5_k9ssrs) from [TechWorld with Nana](https://www.techworld-with-nana.com/).

## K8S Setup

First we need a working K8S setup

### Create a AKS Cluster

**Prequesite:**

We are going to use the [30 Day Free Azure Account](https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account) for this tutorial, after creating it we need to setup [azure-cli](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) on our local machine.
After installing run `az login` to login to your azure account and run `az aks install-cli` to install aks `kubectl`.
Later on we'll also need a domain we can point to your K8S ingress, I use my own but you could easily use [duckdns](https://www.duckdns.org/) for this.

First we need to define some variables that we are going to use later on:

```shell
# Resource group name
RESOURCE_GROUP="rg-aks-argocd-demo"
# Azure Region we want to use
LOCATION="westeurope"
# Azure K8S cluster name
AKS_NAME="aks-argocd-demo"
# Node size and count, we'll keep it as small as possible
# standard_a2_v2: 2 Cores, 4GB RAM
NODE_SIZE="standard_a2_v2"
NODE_COUNT=1
```

Now we can create a resource group:

```shell
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

Since this is a new subscription we need to register the following resource providers for our K8S cluster to work:

```shell
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Network
```

This may take a couple of minutes, check the status using:

```shell
az provider list -o json | jq '.[] | select((.namespace=="Microsoft.ContainerService") or (.namespace=="Microsoft.OperationalInsights") or (.namespace=="Microsoft.Network")) | "\(.namespace): \(.registrationState)"' -r
```

And bow we can create our AKS cluster:

```shell
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME \
  --node-vm-size $NODE_SIZE \
  --node-count $NODE_COUNT \
  --enable-managed-identity \
  --enable-aad \
  --enable-azure-rbac \
  --network-plugin azure \
  --network-policy azure \
  --enable-addons monitoring \
  --generate-ssh-keys
```

Wait for it to be ready...

Congratualtions, we created a Azure AKS Cluster 🎉

To continue we need to add the the context to `kubectl`:

```shell
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME
```

To admisistrate the cluster we need to add our user to the "Azure Kubernetes Service RBAC Cluster Admin" Role, since we got Azure AD + Azure RBAC enabled:

```shell
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create \
  --assignee $USER_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $(az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query id -o tsv)
```

wait a minute for the new role to propergate.

Now we can verify that we are connteced:

```shell
kubectl get nodes
kubectl get pods -A
```

The cluster is now ready for deployments.

### Create a Ingress

But we can deploy ArgoCD we need create a Ingress to reach it securely. Here we are using [traefik](https://traefik.io/traefik).
To setup traefik we first need to install [helm](https://helm.sh/docs/intro/install/#through-package-managers).

And to automate handling of TLS certs we will also need [cert-manager](https://cert-manager.io/). So this is where we are going to start

#### cert-manager

Let's create a name space and deploy cert-manager:

```shell
kubectl create namespace cert-manager
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Now we need to configure it. Namely we need to set issuer for let's encrypt (E-Mail) and the solver (http01). Create a new file called `cluster-issuer-staging.yaml` (Put in your E-Mail address):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: postmaster@stack-dev.de  # replace!
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: traefik
```

Apply the config:

```shell
kubectl apply -f cluster-issuer-staging.yaml
```

Ok, now let's deploy our traefik ingress

#### Traefik

Add the helm repo, create a namespace and deploy traefik:

```shell
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik

helm install traefik traefik/traefik \
  --namespace traefik \
  --set ingressClass.enabled=true \
  --set ingressClass.isDefaultClass=true \
  --set service.type=LoadBalancer
```

After successfull deployment we need our public IP address:

```shell
kubectl get svc -n traefik traefik
```

Create a new DNS A-Record with the value of `EXTERNAL-IP`, for me this will be: `argocd.demo.k8s.stack-dev.de`
Test it with:

```shell
nslookup argocd.demo.k8s.stack-dev.de
```

OK now let's create a staging config first to test our setup, to avoid the let's encrypt rate limits, create a new file called `argocd-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - argocd.demo.k8s.stack-dev.de # replace!
    secretName: argocd-server-tls
  rules:
  - host: argocd.demo.k8s.stack-dev.de # replace!
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

And now we apply our staging config:

```shell
kubectl apply -f argocd-ingress.yaml
```

And check our cert status:

```shell
kubectl get certificate -n argocd -w
```

Now we can deploy ArgoCD

### Deploying ArgoCD

To deploy ArgoCD in our cluster create a file called `argocd-install.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.insecure: "true"
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Create a namespace and deploy it:

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f argocd-install.yaml
```

You can watch you deployment using:

```shell
kubectl get pods -n argocd -w
```

Once all services are `Running`, checkout the domain you set earlier, e.g.:
https://argocd.demo.k8s.stack-dev.de

**BEWARE:** You will get a certificate warning, since we are still using a staging certificate!

You can get the ArgoCD admin password with:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

You can now login and change the password.

### Finalizing

Finnaly we can replace the staging TLS certificate with a production certificate.

Therefor edit: `argocd-ingress.yaml` and change `cert-manager.io/cluster-issuer` from `"letsencrypt-staging"` to `"letsencrypt"`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - argocd.demo.k8s.stack-dev.de # replace!
    secretName: argocd-server-tls
  rules:
  - host: argocd.demo.k8s.stack-dev.de # replace!
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

Apply the new config:

```shell
kubectl apply -f argocd-ingress.yaml
```

Visit your domain again, it should now have a valid TLS certificate.

## Using ArgoCD

Now we can use ArgoCD to deploy our first App, for testing we'll be using the [ArgoCD Guestbook Example](https://github.com/argoproj/argocd-example-apps/tree/master/kustomize-guestbook).

First let's create a new DNS A-Record for our guestbook: guestbook.demo.k8s.stack-dev.de

It should also point to `EXTERNAL-IP`.

After creating the entry we can configure ArgoCD.
