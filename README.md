# Ecommerce-User-service
## Micro-Sevices
1. Publisher/Subscriber Architecture
2. Event Streaming Architecture
---
Event-Driven Architecture Benefits
* Decoupling
* Immutability
* Persistability
* Scalability
* Real-time Workflows
* Simplified Auditing
---
Event-Driven Architecture Challenges
1. Performance
2. Eventual Consistency
3. Complexity

## Architecture
## Prerequistes
* Dockerhub
* Go
* Docker
* Kubectl
* Minikube
* Helm
* Chatgpt

## Getting started

Clone the repo vis SSH or HTTPS
```bash
#SSH
git clone git@github.com:Thab310/ecommerce-users.git

#HTTPS
git clone git@github.com:Thab310/ecommerce-users.git

````
src
```bash
cd src/
go mod init <module>
go mod tidy
go get github.com/lib/pq #database driver for postgres
```
Create a docker hub public repository

build and push docker image:

```bash
docker build -t ecommerce-users .
docker image ls
docker login --username thabelo
docker tag <image-id> thabelo/ecommerce-users:0.05
docker push thabelo/ecommerce-users:0.05
cd ..
```

Start minikube local k8s cluster (comprised of 1 node which deploys both master pods and worker pods in 1 node )

```bash
sudo systemctl start docker # or just run your docker desktop if that is what you're running
minikube start --driver=docker
```
encode your secrets and store them as base64 encoded (Note this is not the same as encrption)
```bash
echo <secret> | base64
```
> [!TIP]
Running databases on k8s using deployments and stateful-sets is not the best way to run databases, we have a much better alternative which is running them through controllers like (ClouNativePG).

For more information on CloudNativePG, please check out their beautiful github documentation:
- [Getting started with CloudNativePG ](https://github.com/cloudnative-pg/cloudnative-pg/blob/main/docs/src/quickstart.md)

Adopting `Operators`, `Custom Resource Definitions` (CRDs) and `Controllers` allows us to further extend the vanilla way of building on k8s

k8s Controllers
* CloudNativePG
* Prometheus
* Grafana


### Add helm repositories 
```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
```
```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts 
```

### Install helm charts
> [!WARNING]
If you want to enable monitoring in cnpg you must install prometheus operator CRDs. Hence we start with prometheus operator. This command will install `Prometheus`, `Grafana` and `Alert Manager`. These resources will use values from the `kube-stack-config.yaml` file

```bash
helm upgrade --install prometheus-community \
  --namespace observability \
  --create-namespace \
  -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/docs/src/samples/monitoring/kube-stack-config.yaml \
  prometheus-community/kube-prometheus-stack \
  --wait
```
* From the Prometheus installation, you will have the Prometheus Operator watching for any `PodMonitor`.
* The Grafana installation will be watching for a Grafana dashboard `ConfigMap`.


Install CloudNativePG Controller
```bash
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  --set monitoring.grafanaDashboard.create=true \
  cnpg/cloudnative-pg \
  --wait
```

Verify creation of cnpg deployment
```bash
kubectl get deployment -n cnpg-system
```

### Deploy postgresql cluster
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/database-cluster.yaml
```
Addresses to communicate with our databases, use the one that ends with `-rw`
```bash
kubectl get svc -n users
```
Secrets that our app should use to establish database connection, use the one that ends with `app`:
```bash
kubectl get secrets -n users
```

### Monitoring with Prometheus
run this command to view grafana dash board that was deployed using a helper manifest that is specific to the ClouNativePG project
```bash
kubectl port-forward svc/prometheus-community-kube-prometheus 9090 -n observability
```
Then access the Prometheus console locally at: `http://localhost:9090/`

You can now define some alerts by creating a prometheusRule:

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/main/docs/src/samples/monitoring/prometheusrule.yaml
```  
You should see the default alerts now:

```bash
kubectl get prometheusrules 
```
In The Prometheus dashboard, you can click on the Alerts menu to see the alerts we just installed

### Monitoring with Grafana
In our "plain" installation, Grafana is deployed with no predefined dashboards.

```bash
kubectl port-forward svc/prometheus-community-grafana 3000:80 -n observability
```
Access Grafana locally at `http://localhost:3000/` 

CloudNativePG provides a default dashboard for Grafana as part of the official Helm chart. You can also download the grafana-dashboard.json file and manually importing it via the GUI.

> [!NOTE] 
Use `admin` as the username and `prom-operator` as the password (defined in kube-stack-config.yaml).

### Create table in postgres pod
```bash
 kubectl exec -it users-postgres-1 -n users -- bash
 psql app
 \du
```
Run this command to create the table
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);
```
run this command to show the table
```sql
\dt 
```
```sql
\q
```
```bash
exit
```


### Connecting my app to the database
We are connecting our app to the postressql database through its service

1. users-postgres-rw = name of the cnpg k8s service
2. user = user (from the cnpg secret created by the cnpg cluster)
3. dbname = name of the database (also from the secret created by the cnpg cluster)

```yaml
env:
- name: DATABASE_URL
  value: "postgres://$(user):$(password)@users-postgres-rw:5432/$(dbname)?sslmode=disable"
envFrom:
- secretRef:
    name: users-postgres-app #secret created by cnpg cluster
```

```bash
kubectl apply -f k8s/user-deployment.yaml
```