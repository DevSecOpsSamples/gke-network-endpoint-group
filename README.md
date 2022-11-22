# Sample project for GKE Network Endpoint Group(NEG)

[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-network-endpoint-group&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-network-endpoint-group) [![Lines of Code](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-network-endpoint-group&metric=ncloc)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-network-endpoint-group)

The sample project to compare Network Endpoint Group(NEG)/ClusterIP and Node Port of GKE.

- [app.py](app/app.py)
- [neg-ingress-api-template.yaml](app/neg-ingress-api-template.yaml)
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
kubectl create namespace neg-ingress-api
kubectl create namespace loadbalancer-type-api

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

## Deploy neg-ingress-api

Create and deploy K8s Deployment, Service, HorizontalPodAutoscaler, Ingress, and GKE BackendConfig using the template files.
It may take around 5 minutes to create a load balancer, including health checking.

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" neg-ingress-api-template.yaml > neg-ingress-api.yaml
cat neg-ingress-api.yaml

kubectl apply -f neg-ingress-api.yaml -n neg-ingress-api
```

[neg-ingress-api-template.yaml](app/neg-ingress-api-template.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: neg-ingress-api
  annotations:
    app: neg-ingress-api
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"default": "neg-ingress-api-backend-config"}'
spec:
  selector:
    app: neg-ingress-api
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: neg-ingress-api-ingress
  annotations:
    app: neg-ingress-api
    kubernetes.io/ingress.class: gce
spec:
  rules:
  - http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: neg-ingress-api
                port:
                  number: 8000
---
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: neg-ingress-api-backend-config
spec:
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 10
    healthyThreshold: 1
    unhealthyThreshold: 3
    port: 8000
    type: HTTP
    requestPath: /ping
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=neg-ingress-api -n neg-ingress-api

kubectl describe pods -n neg-ingress-api

kubectl describe svc -n neg-ingress-api

kubectl get ingress neg-ingress-api-ingress -n neg-ingress-api
```

### Screenshots

- Services & Ingress > SERVICES

    https://console.cloud.google.com/kubernetes/discovery

    ![neg-ingress](./screenshots/neg-ingress-1-services.png?raw=true)

- Services & Ingress > Service details > OVERVIEW

    ![neg-ingress](./screenshots/neg-ingress-2-service-overview.png?raw=true)

- Services & Ingress > Service details > DETAILS

    ![neg-ingress](./screenshots/neg-ingress-3-service-details.png?raw=true)

- Services & Ingress > INGRESS > DETAILS

    https://console.cloud.google.com/kubernetes/ingresses
    ![neg-ingress](./screenshots/neg-ingress-4-ingress-details.png?raw=true)

- Network services > Load balancing > LOAD BALANCERS

    https://console.cloud.google.com/net-services/loadbalancing/list/loadBalancers

    ![loadbalancing](./screenshots/neg-ingress-lb-1.png?raw=true)

- Network services > Load balancing > BACKENDS

    https://console.cloud.google.com/net-services/loadbalancing/list/backends

    ![loadbalancing](./screenshots/neg-ingress-lb-2-backend.png?raw=true)

    ![loadbalancing](./screenshots/neg-ingress-lb-3-backend-details.png?raw=true)

## Deploy loadbalancer-type-api

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" loadbalancer-type-api-template.yaml > loadbalancer-type-api.yaml
cat loadbalancer-type-api.yaml

kubectl apply -f loadbalancer-type-api.yaml -n loadbalancer-type-api
```

Create a Service without Ingress and define the `nodePort: 30000`.

[loadbalancer-type-api-template.yaml](app/loadbalancer-type-api-template.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-type-api
  annotations:
    app: loadbalancer-type-api
spec:
  selector:
    app: loadbalancer-type-api
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8000
      nodePort: 30000
      protocol: TCP
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=loadbalancer-type-api -n loadbalancer-type-api

kubectl describe pods -n loadbalancer-type-api

kubectl describe svc -n loadbalancer-type-api > svc.txt
```

---

## Cleanup

```bash
kubectl delete -f app/neg-false-api.yaml -n neg-false-api
kubectl delete -f app/neg-ingress-api.yaml -n neg-ingress-api

gcloud container clusters delete sample-cluster
```

## References

- [Cloud Load Balancing > Documentation > Guides > Serverless network endpoint groups overview](https://cloud.google.com/load-balancing/docs/negs/serverless-neg-concepts)

- [Cloud Load Balancing > Documentation > Guides > Container-native load balancing through Ingress](https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing)

- [Google Kubernetes Engine (GKE) > Documentation > Guides > GKE Ingress for HTTP(S) Load Balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress)

- [Google Kubernetes Engine (GKE) > Documentation > Guides > Container-native load balancing through standalone zonal NEGs](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg)
