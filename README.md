# Python sample project for GKE

[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-network-endpoint-group&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-network-endpoint-group) [![Lines of Code](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-network-endpoint-group&metric=ncloc)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-network-endpoint-group)

The sample project to compare Network Endpoint Group(NEG)/ClusterIP and Node Port of GKE.

- [app.py](app/app.py)
- [neg-true-api-template.yaml](app/neg-true-api-template.yaml)
- [neg-false-api-template.yaml](app/neg-false-api-template.yaml)

---

## Prerequisites

### Installation

- [Install the gcloud CLI](https://cloud.google.com/sdk/docs/install)
- [Install kubectl and configure cluster access](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl)

### Set environment variables

```bash
PROJECT_ID="sample-project" # replace with your project
COMPUTE_ZONE="us-central1"
```

### Set GCP project

```bash
gcloud config set project ${PROJECT_ID}
gcloud config set compute/zone ${COMPUTE_ZONE}
```

---

## Create a GKE cluster

Create an Autopilot GKE cluster. It may take around 9 minutes.

```bash
gcloud container clusters create-auto sample-cluster --region=${COMPUTE_ZONE}
gcloud container clusters get-credentials sample-cluster
```

```bash
kubectl create namespace neg-true-api
kubectl create namespace neg-false-api
```

---

## Build and push to GCR

```bash
cd ./app
docker build -t python-ping-api . --platform linux/amd64
docker tag python-ping-api:latest gcr.io/${PROJECT_ID}/python-ping-api:latest

gcloud auth configure-docker
docker push gcr.io/${PROJECT_ID}/python-ping-api:latest
```

## Deploy neg-true-api

Create and deploy K8s Deployment, Service, HorizontalPodAutoscaler, Ingress, and GKE BackendConfig using the template files.
It may take around 5 minutes to create a load balancer, including health checking.

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" neg-true-api-template.yaml > neg-true-api.yaml
cat neg-true-api.yaml

kubectl apply -f neg-true-api.yaml -n neg-true-api
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=neg-true-api -n neg-true-api

kubectl describe pods -n neg-true-api

kubectl get ingress neg-true-api-ingress -n neg-true-api
```

## Deploy neg-false-api

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" neg-false-api-template.yaml > neg-false-api.yaml
cat neg-false-api.yaml

kubectl apply -f neg-false-api.yaml -n neg-false-api
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=neg-false-api -n neg-false-api

kubectl describe pods -n neg-false-api

kubectl get ingress neg-false-api-ingress -n neg-false-api
```

NOTE: If you set `spec.type` as ClusterIP, error occurs like the below:

![loadbalancer](./screenshots/neg-false-ingress-error.png?raw=true)

```bash
apiVersion: v1
kind: Service
metadata:
  name: neg-false-api
  annotations:
    app: neg-false-api
    cloud.google.com/neg: '{"ingress": false}'
    cloud.google.com/backend-config: '{"default": "neg-false-api-backend-config"}'
spec:
  selector:
    app: neg-false-api
  type: ClusterIP # expected "NodePort" or "LoadBalancer"
  ports:
    - port: 8000
      targetPort: 8000
      protocol: TCP
```

---

## Screenshots

![loadbalancer](./screenshots/ingress-1-details.png?raw=true)

![loadbalancer](./screenshots/ingress-2-logs.png?raw=true)

![loadbalancer](./screenshots/ingress-3-event.png?raw=true)

---

## Cleanup

```bash
kubectl delete -f app/neg-false-api.yaml -n neg-false-api
kubectl delete -f app/neg-true-api.yaml -n neg-true-api

gcloud container clusters delete sample-cluster
```

## References

- [Cloud Load Balancing > Documentation > Guides > Serverless network endpoint groups overview](https://cloud.google.com/load-balancing/docs/negs/serverless-neg-concepts)

- [Cloud Load Balancing > Documentation > Guides > Container-native load balancing through Ingress](https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing)

- [Google Kubernetes Engine (GKE) > Documentation > Guides > GKE Ingress for HTTP(S) Load Balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress)

- [Google Kubernetes Engine (GKE) > Documentation > Guides > Container-native load balancing through standalone zonal NEGs](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg)
