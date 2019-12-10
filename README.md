# Running Disciple Tools on Kubernetes

This guide provides instructions on how to get Disciple Tools running using the Helm Wordpress chart. 

### Step by step 

#### Install Helm

```
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account=tiller
```

#### Install ingress controller

```
# externalTrafficPolicy=local allows cluster to see real IP so we can apply the ACL
helm install --name ingress stable/nginx-ingress --set controller.service.externalTrafficPolicy=Local
kubectl get svc ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[0].ip}"
```

#### Set A record to ingress controller IP

```
Use the IP from the previous step to create an A record for your site. Check with your DNS provider for 
instructions specific for their service.
```

#### Install cert-manager (Jetstack version)

```
helm repo add jetstack https://charts.jetstack.io
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.8/deploy/manifests/00-crds.yaml
```

Then

```
kubectl create -f letsencrypt-prod.yml
```

Once that has been applied install cert-manager using Helm

```
helm install --name cert-manager --namespace cert-manager --version v0.8.1 jetstack/cert-manager \
   --set ingressShim.defaultIssuerName=letsencrypt-prod \
   --set ingressShim.defaultIssuerKind=ClusterIssuer
```
With cert-manager installed it will automatically request and sign free SSL certificates from 
letsencrypt and will renew certs when they are ready to expire.

#### Deploy Wordpress using helm


##### Create configmap for custom-htaccess

`k create configmap custom-htaccess --from-file wordpress-htaccess.conf`

The file must be named accordingly otherwise the key will be incorrect

```
helm install --name dtools stable/wordpress -f values-production.yaml
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

#### Links

[Limiting access by IP](https://medium.com/@maninder.bindra/using-nginx-ingress-controller-to-restrict-access-by-ip-ip-whitelisting-for-a-service-deployed-to-bd5c86dc66d6)
[Bitnami cert-manager tutorial](https://docs.bitnami.com/kubernetes/how-to/secure-kubernetes-services-with-ingress-tls-letsencrypt/)
[Limit access to GKE master](https://cloud.google.com/kubernetes-engine/docs/how-to/authorized-networks?hl=en_GB&_ga=2.225641433.-278575634.1571199508)
