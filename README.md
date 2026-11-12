# Uber deployment

This repository holds the local kind cluster configuration and Kubernetes deployment resources. The cluster has one control-plane node and four workers: three labeled `workload=app` and one labeled `workload=infra`. The infra worker also runs ingress-nginx and exposes HTTP/HTTPS to the host.

## Create the cluster

Run from this repository root:

```sh
kind create cluster --config kind-config.yaml
kubectl get nodes --show-labels
```

Install ingress-nginx on the infra worker:

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f helm/ingress-nginx/values-kind.yaml
```

Application Deployments select `workload: app`; PostgreSQL and RabbitMQ select `workload: infra`. Create the credentials separately, then install the workloads and apply the ingress rules:

```sh
kubectl create namespace uber
kubectl create secret generic uber-postgres-auth --namespace uber \
  --from-literal=POSTGRES_USER=uber \
  --from-literal=POSTGRES_PASSWORD='REPLACE_WITH_STRONG_PASSWORD'
kubectl create secret generic uber-rabbitmq-auth --namespace uber \
  --from-literal=RABBITMQ_DEFAULT_USER=uber \
  --from-literal=RABBITMQ_DEFAULT_PASS='REPLACE_WITH_STRONG_PASSWORD'
helm upgrade --install uber helm/uber-microservices \
  --namespace uber
kubectl apply -f kubernetes/ingress.yaml
```

The rules route `/api/iam`, `/api/user`, and `/api/rider` to the APIs (stripping the prefix), and `/` to the UI.
The chart deploys the `iam`, `user`, `route`, `rider`, and `payment` images from `nishantbhardwaj/microservices-deployment` using matching image tags. The Services use NodePorts `30001`–`30005`, mapped by kind to `localhost:3001`–`localhost:3005`. Recreate the kind cluster after changing `kind-config.yaml` for the port mappings to take effect. `route` and `payment` are available on those local ports but are not exposed through the ingress. The UI remains a separate deployment.
PostgreSQL and RabbitMQ run as single-replica StatefulSets with 1Gi persistent volumes and internal Services at `postgres:5432` and `rabbitmq:5672`. RabbitMQ's management UI is internal at `rabbitmq:15672`. Configure application connection settings in the service images to use these names and the credentials above; the chart does not assume application-specific environment variable names.

Add `127.0.0.1 uber.local` to your hosts file to browse the UI at `http://uber.local`. Do not commit credentials; create Kubernetes Secrets separately.
