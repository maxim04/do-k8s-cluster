# Digitalocean kubernetes cluster setup

## Steps
Get $200 in credits

[![DigitalOcean Referral Badge](https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%201.svg)](https://www.digitalocean.com/?refcode=24070b3efa68&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
### Pre-reqs
- Custom domain in order to add subdomains
- Kubernetes cluster in digitalocean

### Install nginx ingress load balancer
- choose hostname for load balancer and set in `nginx-ingress-values.yml`, e.g. lb.example.com
- install nginx ingress controller:
  ```
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  ```
  ```
  helm repo update ingress-nginx
  ```
  ```
  helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace -f nginx-ingress-values.yml
  ```
- wait for load balancer to fully setup in digitalocean
- add 2 loadbalancer hostname DNS A records: 
    - lb -> loadbalancer ip
    - www.lb -> loadbalancer ip

### Install cert manager & cluster issuer for issuing ssl certs:
- set your email in `cert-issuer.yml`, it must be a valid email.
- install manager & cert issuer:
  ```
  helm repo add jetstack https://charts.jetstack.io
  ```
  ```
  helm repo update jetstack
  ```
  ```
  helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace -f cert-manager-values.yml
  ```
  ```
  kubectl apply -f cert-issuer.yml
  ```

### Install private container registry using digital ocean spaces as storage

- create 2 hostname DNS CNAME records for registry to point to your loadbalancer domain: 
    - registry -> lb.example.com
    - www.registry -> lb.example.com
- set actual registry hostname in `docker-registry-values.yml` e.g. registry.example.com
- create spaces bucket in digital ocean
- set `s3.region`, `s3.regionEndpoint` & `s3.bucket` in `docker-registry-values.yml`
- create access & secret key for bucket
- set `secrets.s3.accessKey` & `secrets.s3.secretKey` in `docker-registry-values.yml`
- create registry username & password using: 
  ```
  docker run --rm -ti xmartlabs/htpasswd myuser1 password1 >> htpasswd_file
  ```
- replace `username:password` under `secrets.htpasswd` with contents of `htpasswd_file` in `docker-registry-values.yml`
- create container registry:
  ```
  helm repo add twuni https://helm.twun.io
  ```
  ```
  helm repo update twuni
  ```
  ```
  helm install docker-registry twuni/docker-registry -f docker-registry-values.yml --create-namespace --namespace docker-registry
  ```

- TODO Later: create registry auth secret in namespace where container images will be used:
  ```
  kubectl create secret docker-registry regcred --docker-server=registry.example.com --docker-username=myuser1 --docker-password=password1 -n my-app-namespace
  ```

### Install argocd for continuous deployment

- create 2 hostname DNS CNAME records for argocd to point to your loadbalancer domain: 
    - argocd -> lb.example.com
    - www.argocd -> lb.example.com
- set actual registry hostname in `argocd-ingress-http.yml` e.g. argocd.example.com
- install argocd:
  ```
  kubectl create namespace argocd
  ```
  ```
  kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
  ```
  ```
  kubectl apply -f argocd-ingress-http.yml
  ```
- get admin password: 
  ```
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  ```
- navigate to argocd.example.com and login with username: admin, and apssword from previous step.
