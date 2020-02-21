# Running Disciple Tools on Kubernetes

This guide provides instructions on how to get Disciple Tools running using the Helm Wordpress chart. 

### Step by step 

#### Install Helm
Updated for Helm v3

```
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

#### Install ingress controller

externalTrafficPolicy=local allows cluster to see real IP so we can apply the ACL
```
helm install  ingress stable/nginx-ingress --set controller.service.externalTrafficPolicy=Local
kubectl get svc ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

#### Set A record to ingress controller IP

```
Use the IP from the previous step to create an A record for your site. Check with your DNS provider for 
instructions specific for their service.
```

#### Install cert-manager (Jetstack version)

```
kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.13.0/deploy/manifests/00-crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager \
  --namespace cert-manager \
  --version v0.13.0 \
  --set ingressShim.defaultIssuerName=letsencrypt-prod \
  --set ingressShim.defaultIssuerKind=ClusterIssuer \
  --set ingressShim.defaultIssuerGroup=cert-manager.io \
  jetstack/cert-manager
```

##### Then create the issuer 

```
kubectl create -f letsencrypt-prod.yml
```

With cert-manager installed it will automatically request and sign free SSL certificates from 
letsencrypt and will renew certs when they are ready to expire.

#### Deploy Wordpress using helm


##### Create configmap for custom-htaccess

`k create configmap custom-htaccess --from-file wordpress-htaccess.conf`

The file must be named accordingly otherwise the key will be incorrect

```
helm install  dtools stable/wordpress -f values-production.yaml
```


You should then be able to access your site using the FQDN specified earlier. All that remains is to install the
Disciple Tools theme and customise the site to your liking.

#### Operational tasks

### Backups

The SQL database can be backed up by connecting to the database and dumping the database to a local file. 

```
kubectl get svc
```

Gives us a list of services. In the next step you will use kubectl to remotely connect to the MySQL / MariaDB :

```
kubectl port-forward service/dtools-mariadb 3306:3306
```

This will map the database to port 3306 on your local machine. Now we can use `mysqldump` to dump the database :

```
mysqldump -P 3306 -h 127.0.0.1 -u bn_wordpress -p bitnami_wordpress > dtools-$(date +%F).sql
```

During the backup Kubernetes will automatically place the website into maintenance mode while the database 
is dumped and will return it to an operational state afterward.

#### TODO

1. Logs should be monitored
2. This should be thoroughly tested before live data is used. App lifecycle (updates and backup lifecycle specifically should be tested).
3. Configure backup as a kubernetes Cronjob

