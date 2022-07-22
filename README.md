sigstore-install
================

My notes to install sigstore components on a local environment (minikube).

Credits to [Andrew Block](https://github.com/sabre1041), his article [Scaffolding Sigstore](https://medium.com/sigstore/scaffolding-sigstore-e893eb962f22) that is the base and contains more detailed explanations.

## Getting started

### Local tools

* minikube
* kubectl
* helm
* jq
* curl
* openssl

### Sigstore deployment

Start minikube and set the OIDC provider

```
minikube start --extra-config=apiserver.service-account-issuer=https://kubernetes.default.svc
```

Deploy a Nginx ingress controller

```
minikube addons enable ingress
```

Create a `ClusterRoleBinding` to allow anonymous query of the OIDC well-known configuration

```
kubectl create -f deploy/crb-oidc-reviewer.yml
```

Install [cert-manager](https://cert-manager.io/docs/) to automate the creation of TLS certificates to encrypt ingress traffic

```
helm repo add jetstack https://charts.jetstack.io

helm repo update

helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.8.0 \
  --set installCRDs=true
```

Create a `ClusterIssuer` to generate self-signed certificates for our `Ingress`

```
kubectl create -f deploy/clusterissuer-selfsigned-issuer.yml
```

**Deploy sigstore**

```
helm repo add sigstore https://sigstore.github.io/helm-charts

helm repo update

helm upgrade -i scaffold sigstore/scaffold \
  --set ctlog.createctconfig.backoffLimit=30 \
  --set fulcio.server.ingress.http.hosts[0].host=fulcio.$(minikube ip).nip.io \
  --set fulcio.server.ingress.http.hosts[0].path=/ --set rekor.server.ingress.hosts[0].path=/ \
  --set fulcio.server.ingress.tls[0].hosts[0]="fulcio.$(minikube ip).nip.io" \
  --set fulcio.server.ingress.tls[0].secretName=fulcio-tls \
  --set fulcio.server.ingress.http.annotations."cert-manager\.io\/cluster-issuer"=selfsigned-issuer \
  --set fulcio.server.ingress.http.annotations."cert-manager\.io\/common-name"="fulcio.$(minikube ip).nip.io" \
  --set rekor.server.ingress.hosts[0].host=rekor.$(minikube ip).nip.io \
  --set rekor.server.ingress.tls[0].hosts[0]="rekor.$(minikube ip).nip.io" \
  --set rekor.server.ingress.tls[0].secretName=rekor-tls \
  --set rekor.server.ingress.annotations."cert-manager\.io\/cluster-issuer"=selfsigned-issuer \
  --set rekor.server.ingress.annotations."cert-manager\.io\/common-name"="rekor.$(minikube ip).nip.io" \
  -n sigstore \
  --create-namespace
```

1. set the number of attempts for the ctlog configuration job \
2. configure fulcio with minikube ip, path, secret and selfsigned-issuer \
3. configure rekor with minikube ip, secret and selfsigned-issuer \

This chart is going to create 4 namespaces:
* ctlog-system
* fulcio-system
* rekor-system
* trillian-system
