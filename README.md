# Uber deployment

This repository contains the kind cluster and Kubernetes deployment configuration for the User, Route, Rider, and Payment backend services. It does not deploy IAM or a UI. The application images still require a shared `JWT_SECRET` to start.

The local kind cluster has one control-plane node and four workers: three labeled `workload=app` and one labeled `workload=infra`. The four APIs schedule on app workers. PostgreSQL, RabbitMQ, and ingress-nginx schedule on the infra worker.

## Create the cluster

Run from this repository root:

```sh
kind create cluster --config kind-config.yaml
kubectl get nodes --show-labels
```

Kind maps ports 80/443 to the ingress worker and localhost ports 3002-3005 to the API NodePorts; port 3001 is unused after removing IAM. These mappings are fixed at cluster creation; recreate the cluster if they change.

## Install ingress-nginx

```sh
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f helm/ingress-nginx/values-kind.yaml
```

## Create credentials and deploy

Use URL-safe hex passwords because the services build PostgreSQL and AMQP URLs from these values. Keep the actual credentials private and do not commit them.

```sh
kubectl create namespace uber --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic uber-postgres-auth --namespace uber \
  --from-literal=POSTGRES_USER=uber \
  --from-literal=POSTGRES_PASSWORD='REPLACE_WITH_RANDOM_HEX' \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic uber-rabbitmq-auth --namespace uber \
  --from-literal=RABBITMQ_DEFAULT_USER=uber \
  --from-literal=RABBITMQ_DEFAULT_PASS='REPLACE_WITH_RANDOM_HEX' \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic uber-app-auth --namespace uber \
  --from-literal=JWT_SECRET='REPLACE_WITH_A_RANDOM_SECRET' \
  --dry-run=client -o yaml | kubectl apply -f -

helm lint helm/uber-microservices
helm upgrade --install uber helm/uber-microservices --namespace uber
kubectl apply -f kubernetes/ingress.yaml
kubectl delete ingress uber-api -n uber --ignore-not-found
kubectl get pods,services,ingress -n uber -o wide
```

The chart configures the APIs with in-cluster PostgreSQL and RabbitMQ URLs and credentials. It deploys the image tags `user`, `route`, `rider`, `payment`, `postgres`, and `rabbitmq` from `nishantbhardwaj/microservices-deployment`; ensure these images have been pushed and are publicly pullable. For a private Docker Hub repository, create a registry Secret in namespace `uber` and add it under `image.pullSecrets` in `helm/uber-microservices/values.yaml`.

## Access the APIs

Ingress is host-independent: `/` routes to the User API; `/user/*`, `/rider/*`, `/route/*`, and `/payment/*` route to their respective APIs with the prefix stripped. For example, `/user/health` reaches User's `/health`, and `/rider/health` reaches Rider's `/health`. These are API routes, not browser pages; the images do not contain a frontend UI.

Use `http://<machine-ip>/user/health`, `/rider/health`, `/route/health`, or `/payment/health` through ingress-nginx to check each API. User, Route, Rider, and Payment can also be accessed directly on localhost ports 3002-3005 through the mapped NodePorts.

PostgreSQL and RabbitMQ are single-replica StatefulSets with 1Gi persistent volumes. The volumes are for local practice, not backups or high availability.
