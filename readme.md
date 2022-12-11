# Digitalocean kubernetes cluster setup

## Steps
### Pre-reqs
- Custom domain in order to add subdomains
- Kubernetes cluster in digitalocean

### Install nginx ingress load balancer
- add loadbalancer hostname DNS A Record: lb.example.com -> loadbalancer ip
- set actual hostname in `nginx-ingress-values.yml`
- install nginx ingress controller:

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

helm repo update ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace -f nginx-ingress-values.yml
```

### Install cert manager & cluster issuer for issuing ssl certs:
- set your email in `cert-issuer.yml`
- install manager & cert issuer:
```
helm repo add jetstack https://charts.jetstack.io

helm repo update jetstack

helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace -f cert-manager-values.yml

kubectl apply -f cert-issuer.yml
```

### Install private container registry using digital ocean spaces as storage
- create hostname DNS CNAME entry for registry: registry.example.com -> lb.example.com
- set actual registry hostname in `docker-registry-values.yml`
- create spaces bucket in digital ocean
- set `s3.region`, `s3.regionEndpoint` & `s3.bucket` in `docker-registry-values.yml`
- create access & secret key for bucket
- set `secrets.s3.accessKey` & `secrets.s3.accessKey` in `docker-registry-values.yml`
- create registry username & password using: `docker run --rm -ti xmartlabs/htpasswd myuser1 password1 >> htpasswd_file`
- replace `username:password` under `secrets.htpasswd` with contents of `htpasswd_file` in `docker-registry-values.yml`
- create container registry:
```
helm repo add twuni https://helm.twun.io

helm repo update

helm install docker-registry twuni/docker-registry -f docker-registry-values.yml --create-namespace --namespace docker-registry
```
- create registry auth secret in namespace where container images will be used:
```
kubectl create secret docker-registry regcred --docker-server=registry.example.com --docker-username=myuser1 --docker-password=password1 -n my-app-namespace
```

### Install argocd for continuous deployment

- create hostname DNS CNAME entry for argocd: argocd.example.com -> lb.example.com
- set actual registry hostname in `argocd-ingress-http.yml`
- install argocd:
```
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl apply -f argocd-ingress-http.yml
```
- get admin password: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo`
- navigate to argocd.example.com and login with username: admin, and apssword from previous step.