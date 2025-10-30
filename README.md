# Azure AKS - ArgoCD Tutorial

In this tutorial, we’ll set up an Azure AKS cluster and deploy Traefik, ArgoCD, and an example application on it.

## K8S Setup

First, we need a working Kubernetes setup.

### Create an AKS Cluster

**Prerequisites:**

We’ll use the [30-Day Free Azure Account](https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account) for this tutorial. After registering, set up the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) on your local machine.
After installation, run `az login` to log in to your Azure account, and `az aks install-cli` to install the AKS `kubectl` CLI tool.

Later, we’ll also need a domain that we can point to our Kubernetes ingress. I’ll use my own, but you can easily use [DuckDNS](https://www.duckdns.org/) instead.

Let’s define some variables that we’ll use throughout the setup:

```shell
# Resource group name
RESOURCE_GROUP="rg-aks-argocd-demo"
# Azure region we want to use
LOCATION="westeurope"
# Azure AKS cluster name
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

Since this is a new subscription, we need to register the following resource providers for our Kubernetes cluster to work:

```shell
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Network
```

This may take a few minutes. Check the registration status with:

```shell
az provider list -o json | jq '.[] | select((.namespace=="Microsoft.ContainerService") or (.namespace=="Microsoft.OperationalInsights") or (.namespace=="Microsoft.Network")) | "\(.namespace): \(.registrationState)"' -r
```

Now we can create our AKS cluster:

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

Wait for it to finish provisioning.

Congratulations — we’ve created a Kubernetes cluster 🎉

Next, we need to add the context to `kubectl`:

```shell
az aks get-credentials \
  --resource-group $RESOURCE_GROUP \
  --name $AKS_NAME
```

Since we’ve enabled Azure AD and Azure RBAC, we also need to add our user to the **Azure Kubernetes Service RBAC Cluster Admin** role:

```shell
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create \
  --assignee $USER_ID \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --scope $(az aks show --resource-group $RESOURCE_GROUP --name $AKS_NAME --query id -o tsv)
```

Wait a minute for the role assignment to propagate.

Now verify that we’re connected:

```shell
kubectl get nodes
kubectl get pods -A
```

The cluster is now ready for deployments.

### Create an Ingress

Before we can deploy ArgoCD, we need to set up an ingress to reach it securely. We’ll use [Traefik](https://traefik.io/traefik) for this.
To install Traefik, we first need to install [Helm](https://helm.sh/docs/intro/install/#through-package-managers).

To automate TLS certificate management, we’ll also install [cert-manager](https://cert-manager.io/). Let’s start with that.

#### cert-manager

Create a namespace and deploy cert-manager:

```shell
kubectl create namespace cert-manager
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Next, we need to configure it. Specifically, we’ll set up an issuer for Let’s Encrypt (using your email) and the HTTP-01 solver.
Create a new file called `cluster-issuer-staging.yaml` (replace the email address with your own):

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

Apply the configuration:

```shell
kubectl apply -f cluster-issuer-staging.yaml
```

Now, let’s deploy our Traefik ingress.

#### Traefik

Add the Helm repo, create a namespace, and deploy Traefik:

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

After a successful deployment, we’ll need our public IP address:

```shell
kubectl get svc -n traefik traefik
```

Create a new DNS **A record** with the value of `EXTERNAL-IP`.
Mine is for example: `argocd.demo.k8s.stack-dev.de`

Test it with:

```shell
nslookup argocd.demo.k8s.stack-dev.de
```

Now let’s create a staging configuration to test our setup and avoid Let’s Encrypt rate limits.
Create a new file called `argocd-ingress.yaml`:

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

Apply the staging config:

```shell
kubectl apply -f argocd-ingress.yaml
```

Check the certificate status:

```shell
kubectl get certificate -n argocd -w
```

Now we can deploy ArgoCD.

### Deploying ArgoCD

To deploy ArgoCD, create a file called `argocd-install.yaml`:

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

You can watch the deployment with:

```shell
kubectl get pods -n argocd -w
```

Once all services show as `Running`, open the domain you configured earlier, e.g.:
[https://argocd.demo.k8s.stack-dev.de](https://argocd.demo.k8s.stack-dev.de)

**Note:** You’ll see a certificate warning because we’re still using a staging certificate.

Get the ArgoCD admin password with:

```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode && echo
```

You can now log in and change the password.

### Finalizing

Finally, we’ll replace the staging TLS certificate with a production one.

Edit `argocd-ingress.yaml` and change `cert-manager.io/cluster-issuer` from `"letsencrypt-staging"` to `"letsencrypt"`:

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

Apply the updated configuration:

```shell
kubectl apply -f argocd-ingress.yaml
```

Visit your domain again — it should now have a valid TLS certificate.

## Using ArgoCD

Now we can use ArgoCD to deploy our first application.
For testing, we’ll use the [ArgoCD Guestbook Example](https://github.com/argoproj/argocd-example-apps/tree/master/kustomize-guestbook).

First, create a new DNS **A record** for our guestbook:
`guestbook.demo.k8s.stack-dev.de`

It should point to the same `EXTERNAL-IP`.

After creating the DNS entry, we can proceed to configure ArgoCD.
