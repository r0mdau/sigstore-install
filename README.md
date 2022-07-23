sigstore-install
================

My notes to install sigstore components on a local environment (minikube).

Credits to [Andrew Block](https://github.com/sabre1041), his article [Scaffolding Sigstore](https://medium.com/sigstore/scaffolding-sigstore-e893eb962f22) that is the base and contains more detailed explanations.

## Why Sigstore ?

Do you trust every package you download when running commands like:

```
npm install
yarn install
mvn install
go get -v ./...
composer install
dotnet restore
docker build / docker pull
etc...
```

And if you don't trust them, do you have the time to open the `vendor | node_modules ...` folder to scrutinize each line of code ?

We often respond no and no to those questions, or you have the means to develop everything in the house like big techs or government are able to do.

For most cases, cryptographically signing artifacts and publishing those signatures to a public database is one way to allow everybody to verify that the artifact they download comes from a source you trust.

The sigstore project helps you automate signing artifacts and, on the other hand, verify signatures.

## Getting started

Architecture overview

![arch overview](img/alt_landscapelayout_overview.svg)

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

### Deployment verifications

Copy secrets into the `default` namespace

```
kubectl -n ctlog-system get secrets ctlog-public-key -o yaml | sed 's/namespace: .*/namespace: default/' | kubectl apply -f -

kubectl -n fulcio-system get secrets fulcio-server-secret -oyaml | sed -e 's/namespace: .*/namespace: default/' -e 's/name: .*/name: fulcio-secret/' | kubectl apply -f -
```

Deploy a Docker registry in minikube, with a self-signed certificate

```
kubectl create -f verify/certificate-registry.yml

kubectl create -f verify/service-registry.yml

kubectl create -f verify/deploy-registry.yml
```

Create a `Job` to sign an already available image using Fulcio certificates, the produced signature will be stored in the registry we just deployed, due to the `COSIGN_REPOSITORY` environment variable

```
kubectl create -f verify/job-sign.yml
```

Look the logs of the running container to confirm that the image is successfully signed, sample output

```
$ kubectl logs sign-job-fdtmj

Generating ephemeral keys...
Retrieving signed certificate...
**Warning** Using a non-standard public key for verifying SCT: /var/run/sigstore-root/rootfile.pem
Successfully verified SCT...
tlog entry created with index: 0
Pushing signature to: sigstore
```

Cosign has requested a certificate from Fulcio, signed the image and stored the entry in Rekor.

Now verify the signed content with Cosign. `Job` example

```
kubectl apply -f verify/job-verify.yml
```

Again, look at the pod logs to confirm that the job worked

```
$ kubectl logs verify-job-zb9jj

Verification for ghcr.io/sigstore/scaffolding/checktree@sha256:9ca3cf4d20eb4f6c85929cd873234150d746e69dbb7850914bbd85b97eba1e2f --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The claims were present in the transparency log
  - The signatures were integrated into the transparency log when the certificate was valid
  - Any certificates were verified against the Fulcio roots.
...
```

We can also retrieve the record by querying Rekor API, it is another form of validation

```
curl -s --cacert <(kubectl get secrets -n rekor-system rekor-tls -o jsonpath='{ .data.tls\.crt }' | base64 -d) https://rekor.$(minikube ip).nip.io/api/v1/log/entries?logIndex=0 | jq -r
```

Query to inspect the public key

```
curl -s --cacert <(kubectl get secrets -n rekor-system rekor-tls -o jsonpath='{ .data.tls\.crt }' | base64 -d) https://rekor.$(minikube ip).nip.io/api/v1/log/entries?logIndex=0 | jq -r '.[keys_unsorted[0]].body' | base64 -d | jq -r '.spec.signature.publicKey.content' | base64 -d | openssl x509 -noout -text
```

We can observe in the `Subject Alternative Name` the values associated with our identity token from the Fulcio OIDC.

Kudos \o/ you have a working Sigstore architecture working in minikube. Next stop is to deploy it on production clusters.

The [scaffolding Sigstore project](https://github.com/sigstore/scaffolding/) contains many useful resources.

## Further reading

* Sigstore.dev - [How it works](https://www.sigstore.dev/how-it-works)
* SLSA.dev - [Supply chain threats](https://slsa.dev/spec/v0.1/index)
