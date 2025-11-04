# Azure AKS - ArgoCD Tutorial

In this tutorial, we’ll set up an Azure AKS cluster and deploy Traefik, ArgoCD, and an example application on it.

## K8S Setup

First, we need a working Kubernetes setup.

### Create an AKS Cluster

**Prerequisites:**

We will need a Azure Account for this tutorial, I you don't have one, you can use the [30-Day Free Azure Account](https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account). After registering, set up the [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) on your local machine and run `az login` to log in to your Azure account. Then setup `kubectl` by running `az aks install-cli`. We will also need [Helm](https://helm.sh/docs/intro/install/#through-package-managers) to be installed.

Later, we’ll also need a domain that we can point to our Kubernetes ingress. I’ll use my own, but you can easily use [DuckDNS](https://www.duckdns.org/) instead.

First we define some variables that we’ll use throughout the setup:

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

If this is a new subscription, we need to register the following resource providers for our Kubernetes cluster to work:

```shell
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Network
```

This may take a few minutes. Check the registration status with:

```shell
az provider list -o json | jq '.[] | select((.namespace=="Microsoft.ContainerService") or (.namespace=="microsoft.insights") or (.namespace=="Microsoft.OperationalInsights") or (.namespace=="Microsoft.Network")) | "\(.namespace): \(.registrationState)"' -r
```

Now we can create a resource group:

```shell
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION
```

And create our AKS cluster:

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

Before we can deploy ArgoCD, we need to set up an ingress to reach it securely. We’ll use [Traefik](https://traefik.io/traefik) for this combined with [cert-manager](https://cert-manager.io/) for automate TLS certificate management.

Let’s start with `cert-manager`.

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

Next, we need to configure it.
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
Mine for example is: [https://argocd.demo.k8s.stack-dev.de](https://argocd.demo.k8s.stack-dev.de)

Test it with:

```shell
nslookup argocd.demo.k8s.stack-dev.de
```

Now we can deploy ArgoCD.

### Deploying ArgoCD

We'll turn off auto redirect to https in ArgoCD since we are using traefik as reverse proxy in front of it, therefore we need to create some files.
First create a folder called `argocd`, inside create two files, `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: argocd

resources:
  - https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

patches:
  - path: argocd-cmd-params-cm-patch.yaml
    target:
      kind: ConfigMap
      name: argocd-cmd-params-cm

```

and a file called `argocd-cmd-params-cm-patch.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
data:
  server.insecure: "true"
```

What we are doing here is patching the official install chart to set the default connection to http using [kustomize](https://github.com/kubernetes-sigs/kustomize).

Create a name space and deploy it:

```shell
kubectl create namespace argocd
kubectl apply -k argocd/
```

You can watch the deployment with:

```shell
kubectl get pods -n argocd -w
```

To reach it we need to create a ingress with a staging configuration for argocd to test our setup and avoid Let’s Encrypt rate limits.
Create a new file called `argocd-ingress.yaml`:


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
    traefik.ingress.kubernetes.io/router.middlewares: traefik-redirect-to-https@kubernetescrd
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

Now we want to add a http to https redirect on our ingress, create a file called: `redirect-middleware.yaml`

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-to-https
  namespace: traefik
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

Apply the staging config:

```shell
kubectl apply -f traefik-redirect-middleware.yaml
kubectl apply -f argocd-ingress.yaml
```

Check the certificate status:

```shell
kubectl get certificate -n argocd -w
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

Create a file called `cluster-issuer-prod.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: postmaster@stack-dev.de  # replace!
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: traefik
```

Apply the production configuration:

```shell
kubectl apply -f cluster-issuer-prod.yaml
```

Then edit `argocd-ingress.yaml` and change `cert-manager.io/cluster-issuer` from `"letsencrypt-staging"` to `"letsencrypt"`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt"
    traefik.ingress.kubernetes.io/router.middlewares: traefik-redirect-to-https@kubernetescrd
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

Apply the updated configuration and delete the old certs:

```shell
# Delete the staging certificate and secret
kubectl delete certificate argocd-server-tls -n argocd
kubectl delete secret argocd-server-tls -n argocd
kubectl apply -f argocd-ingress.yaml
```

Check the certificate status:

```shell
kubectl get certificate -n argocd -w
```

Then visit your domain again, it should now have a valid TLS certificate (You might need to clear the cache).

## Using ArgoCD

Now we can use ArgoCD to deploy our first application.
For testing, we’ll use the [ArgoCD Guestbook Example](https://github.com/argoproj/argocd-example-apps/tree/master/kustomize-guestbook).

First, create a new DNS **A record** for our guestbook:
`guestbook.demo.k8s.stack-dev.de`

It should point to the same `EXTERNAL-IP`.

After creating the DNS entry, we can proceed to configure ArgoCD.

## Delete all

If you are finished testing, you can delete the cluster and local configuration executing the following code.

Remove user, cluster and context from `kubectl` config:

```shell
KUBECTL_USER=$(kubectl config get-users | grep "${RESOURCE_GROUP}_${AKS_NAME}")
kubectl config delete-cluster "$AKS_NAME"
kubectl config unset users."$KUBECTL_USER"
kubectl config delete-context "$AKS_NAME"
```

To remove the whole resource group including the cluster:

```shell
az group delete --name $RESOURCE_GROUP --yes
```
