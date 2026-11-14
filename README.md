# Uber deployment

This repository contains the kind cluster and Kubernetes deployment configuration for the User, Route, Rider, and Payment backend services, plus the static browser UI. IAM is intentionally not deployed.

The local kind cluster has one control-plane node and four workers: three labeled `workload=app` and one labeled `workload=infra`. The four APIs and UI schedule on app workers. PostgreSQL, RabbitMQ, and ingress-nginx schedule on the infra worker.

## Create the cluster

Run from this repository root:

```sh
kind create cluster --config kind-config.yaml
kubectl get nodes --show-labels
```

Kind maps ports 80/443 to the ingress worker. Application services use ClusterIP and are reached through ingress, not exposed as NodePorts.

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
```

From the **application repository root**, rebuild and push the changed images:

```sh
docker build -t nishantbhardwaj/microservices-deployment:user services/user
docker build -t nishantbhardwaj/microservices-deployment:rider services/rider
docker build -t nishantbhardwaj/microservices-deployment:ui ui
docker push nishantbhardwaj/microservices-deployment:user
docker push nishantbhardwaj/microservices-deployment:rider
docker push nishantbhardwaj/microservices-deployment:ui
```

Then from this **deployment repository root**:

```sh
helm lint helm/uber-microservices
helm upgrade --install uber helm/uber-microservices --namespace uber
kubectl apply -f kubernetes/ingress.yaml
kubectl delete ingress uber-rider uber-user -n uber --ignore-not-found
kubectl get pods,services,ingress -n uber -o wide
```

The chart configures the APIs with in-cluster PostgreSQL and RabbitMQ URLs, credentials, and a shared `JWT_SECRET` (required at startup). For this local demo only, it sets `DEMO_AUTH_DISABLED=true` on User and Rider: the services assign fixed demo passenger/rider identities, and the UI has no login. **This bypass is unsafe on public or untrusted networks.** The app source defaults to JWT authentication unless that variable is explicitly set. The chart deploys image tags `user`, `route`, `rider`, `payment`, `ui`, `postgres`, and `rabbitmq` from `nishantbhardwaj/microservices-deployment`; ensure these images have been pushed and are publicly pullable. For a private Docker Hub repository, create a registry Secret in namespace `uber` and add it under `image.pullSecrets` in `helm/uber-microservices/values.yaml`.

## Use the browser app

Ingress is host-independent: `/` serves the passenger dashboard and `/rider` serves the rider dashboard. Browser API calls use `/api/user/*` and `/api/rider/*`; Route and Payment are internal-only services and have no public ingress routes. Route matching and fare calculation remain event-driven; distance, ETA, and final fare appear in ride status/history.

Open `http://<machine-ip>/` for passenger booking or `http://<machine-ip>/rider` for the rider dashboard. User and Rider health checks are available at `/api/user/health` and `/api/rider/health`; Route and Payment are not browser-accessible.

For an end-to-end demo, open both URLs in separate tabs: go online on the Rider page, then request a ride on the passenger page. The rider tab polls for offers and opens a popup; accept, start, and complete the trip there. The passenger ride list updates with matching status, distance/ETA, and fare.

PostgreSQL and RabbitMQ are single-replica StatefulSets with 1Gi persistent volumes. The volumes are for local practice, not backups or high availability.
