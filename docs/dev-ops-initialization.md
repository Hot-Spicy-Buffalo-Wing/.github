# DevOps SetUp
## 1. Make a node
> with TerraForm

We used `CloudFlare` for DNS Provider and `EC2` for Kubernetes(k3s) node.

- vpc terraform: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-infra/blob/main/vpc.tf
- ec2 terraform: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-infra/blob/main/ec2.tf
- dns terraform: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-infra/blob/main/dns.tf

all `dns A record` pointing the **control plane node**

also provide key pair to connect instances
- key-pairs terraform: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-infra/blob/main/key-pairs.tf

ec2 terraform has output so public IPs of instances can be known with `tf output`

## 2. Install K3S

Connecting EC2 instance, install K3S

If you want to connect (access) with kubectl, there is TLS certification issue (hostname not matching)
So, it is required to change the tls configuration after install

### Install control plane

```bash
curl -sFL https://get.k3s.io | sh -
```

#### TLS config
change `/etc/systemd/system/k3s.service`

```
ExecStart=/usr/local/bin/k3s \
    server \
  	'--node-name=k3s-node-a' \
  	'--tls-san=x.x.x.x' \
```

```bash
sudo kubectl -n kube-system delete secrets/k3s-serving
sudo mv /var/lib/rancher/k3s/server/tls/dynamic-cert.json /tmp/dynamic-cert.json
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

### Install other nodes

other nodes can be setup with K3S_TOKEN

in control plane, K3S_TOKEN can be known with `/var/lib/rancher/k3s/server/node-token`

```bash
curl -sfL https://get.k3s.io | K3S_URL="https://myserver:6443" K3S_TOKEN=mynodetoken sh -s -
```

### kubeconfig

is in `/etc/rancher/k3s/k3s.yaml`

## 3. Install ArgoCD

ArgoCD can be deployed with helm chart (is convenient)  
Also, it can be configured with `values.yaml`

We can extract `values.yaml` from helm repository

```bash
brew install helm # in macOS
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm show values argo/argo-cd > argocd/values.yaml
```

ArgoCD values.yaml file: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-gitops/blob/main/argocd/values.yaml

In values.yaml file, setup rbac(role-based access control), dex(SSO) with GitHub

then install ArgoCD
```bash
kubectl create namespace argocd
helm install argo -n argocd argo/argo-cd -f argocd/values.yaml
```

### Connection

#### 1. kubectl port-forward

very simpe way

```bash
kubectl port-forward service/argo-argocd-server -n argocd 8080:443
```

#### 2. Setup Ingress (without TLS configuration)

1. set `server.ingress.enabled` true
    1. set [ssl passthrough](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough)
    2. set [`configs.params."server.insecure"`](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-ssl-termination-at-ingress-controller) true
2. set `name` and `path` of `server.ingress.extraHosts`

after change `values.yaml`, enter below command

```bash
helm upgrade argo -n argocd argo/argo-cd -f argocd/values.yaml
```

to connect, we should know initial credential 

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## 4. Init Repository in ArgoCD

`settings` -> `repositories` -> `connect repo`

Webhook can boost-up deploy speed

`gitops repository` -> `settings` -> `webhooks`
URL: `$ARGO_URL/api/webhook`
content type: application/json
event: just push event

## 5. Cert Manager

https://cert-manager.io/

From ArgoCD 2.6, it supports multi source application, so we can use helm chart and other resources together in one application

Cert Manager Application File: 

First, Add Cert-Manager Application and apply to create application

```bash
kubectl apply -f argocd/cert-manager.yaml -n argocd
```

After create application, ArgoCD traces the changing in gitops repository
(But when change the application manifest itself, it's required to re-apply manifest file)

### 1. Configuring Issuers

we use CloudFlare for DNS Provider, so it can be used as issuers

issuers.yaml: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-gitops/blob/main/cert-manager/clusterissuer.yaml

#### sealed-secrets

we can store secret file in gitops repository safely

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets

helm install sealed-secrets -n kube-system --set-string fullnameOverride=sealed-secrets-controller sealed-secrets/sealed-secrets

brew install kubeseal  # in macOS

echo "API KEY" | k create secret generic cloudflare-api-token-secret --from-file=api-token=/dev/stdin --dry-run=client --output=yaml > cert-manager/cloudflare-secret.yaml

kubeseal -f cert-manager/cloudflare-secret.yaml -w cert-manager/cloudflare-secret.sealed.yaml
```

### 2. Issue Certificate

ArgoCD certificate: https://github.com/Hot-Spicy-Buffalo-Wing/check-out-gitops/blob/main/cert-manager/certs/argocd.yaml

in above example, `reflection` (can be installed with helm) is used to change target namespace

```bash
helm repo add emberstack https://emberstack.github.io/helm-charts
helm repo update
helm upgrade --install reflector emberstack/reflector
```


