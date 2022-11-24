# Sample project for GKE Network Endpoint Group(NEG)

[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-network-endpoint-group&metric=alert_status)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-network-endpoint-group) [![Lines of Code](https://sonarcloud.io/api/project_badges/measure?project=DevSecOpsSamples_gke-network-endpoint-group&metric=ncloc)](https://sonarcloud.io/summary/new_code?id=DevSecOpsSamples_gke-network-endpoint-group)

## Overview

GCP recommend to use the container-native load balancer through Ingress to evenly distribute traffic to Pods.
This project provides sample code to understand about differences among Ingress/ClusterIP, LoadBalancer and Node Port of GKE.

- [ingress-neg-api-template.yaml](app/ingress-neg-api-template.yaml)
- [loadbalancer-type-api.yaml](app/loadbalancer-type-api.yaml)

|                            | Ingress/NEG   |  LoadBalancer     |  NodePort     |
|----------------------------|---------------|-------------------|---------------|
| K8s Service Type           | ClusterIP     | LoadBalancer      | NodePort      |
| Load Balancer Type         | Application Load Balancer | Network Load Balancer | X      |
| Use a NodePort             | X             | O                 | O                 |

```bash
kubectl get svc
```

```bash
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
ingress-neg-api         ClusterIP      10.25.130.238   <none>           8000/TCP       40m
loadbalancer-type-api   LoadBalancer   10.25.131.128   34.133.110.139   80:31000/TCP   31m
nodeport-type-api       NodePort       10.25.129.2     <none>           80:31854/TCP   17m
```

## Objectives

- Learn about difference among Ingress, LoadBalacer, and NodePort on GKE
- Learn about manifests for Deployment, Service, Ingress, BackendConfig, and HorizontalPodAutoscaler.

## Table of Contents

- Create a GKE cluster and namespaces
- Build and push to GCR
- Ingress with Network Endpoint Group(NEG)
    - Object Spec
    - Deploy ingress-neg-api
    - Screenshots
- LoadBalancer Type with NodePort
    - Object Spec
    - Deploy loadbalancer-type-api
    - Screenshots
- Cleanup

---

## Prerequisites

### Installation

- [Install the gcloud CLI](https://cloud.google.com/sdk/docs/install)
- [Install kubectl and configure cluster access](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl)

### Set environment variables

```bash
COMPUTE_ZONE="us-central1"
PROJECT_ID="sample-project" # replace with your project
```

### Set GCP project

```bash
gcloud config set project ${PROJECT_ID}
gcloud config set compute/zone ${COMPUTE_ZONE}
```

---

## 1. Create a GKE cluster

Create an Autopilot GKE cluster. It may take around 9 minutes.

```bash
gcloud container clusters create-auto sample-cluster --region=${COMPUTE_ZONE}
gcloud container clusters get-credentials sample-cluster
```

---

## 2. Build and push to GCR

```bash
cd ./app
docker build -t python-ping-api . --platform linux/amd64
docker tag python-ping-api:latest gcr.io/${PROJECT_ID}/python-ping-api:latest

gcloud auth configure-docker
docker push gcr.io/${PROJECT_ID}/python-ping-api:latest
```

## 3. Ingress with Network Endpoint Group(NEG)

### 3.1 Object Spec

[ingress-neg-api-template.yaml](app/ingress-neg-api-template.yaml):

| Kind    | Element                     | Value             | Description             |
|---------|-----------------------------|-------------------|-------------------------|
| Service | spec.type                   | **ClusterIP**     |                         |
| Service | metadata.annotations        | cloud.google.com/neg: '{"ingress": true}'  |                         |
| Ingress | metadata.annotations        | kubernetes.io/ingress.class: gce           |                         |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-neg-api
  annotations:
    app: ingress-neg-api
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/backend-config: '{"default": "ingress-neg-api-backend-config"}'
spec:
  selector:
    app: ingress-neg-api
  type: ClusterIP
  ports:
    - port: 8000
      targetPort: 8000
      protocol: TCP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-neg-api-ingress
  annotations:
    app: ingress-neg-api
    kubernetes.io/ingress.class: gce
spec:
  rules:
  - http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: ingress-neg-api
                port:
                  number: 8000
---
apiVersion: cloud.google.com/v1
kind: BackendConfig
metadata:
  name: ingress-neg-api-backend-config
spec:
  healthCheck:
    checkIntervalSec: 10
    timeoutSec: 10
    healthyThreshold: 2
    unhealthyThreshold: 5
    port: 8000
    type: HTTP
    requestPath: /ping
```

### 3.2 Deploy ingress-neg-api

Create and deploy K8s Deployment, Service, HorizontalPodAutoscaler, Ingress, and GKE BackendConfig using the template files.
It may take around 5 minutes to create a load balancer, including health checking.

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" ingress-neg-api-template.yaml > ingress-neg-api.yaml
cat ingress-neg-api.yaml

kubectl apply -f ingress-neg-api.yaml
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=ingress-neg-api

kubectl describe pods
```

```bash
kubectl get service ingress-neg-api --output yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    app: ingress-neg-api
    cloud.google.com/backend-config: '{"default": "ingress-neg-api-backend-config"}'
    cloud.google.com/neg: '{"ingress": true}'
    cloud.google.com/neg-status: '{"network_endpoint_groups":{"8000":"k8s1-d0ddc61c-default-ingress-neg-api-8000-276c73a2"},"zones":["us-central1-b","us-central1-c"]}'
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"app":"ingress-neg-api","cloud.google.com/backend-config":"{\"default\": \"ingress-neg-api-backend-config\"}","cloud.google.com/neg":"{\"ingress\": true}"},"name":"ingress-neg-api","namespace":"default"},"spec":{"ports":[{"port":8000,"protocol":"TCP","targetPort":8000}],"selector":{"app":"ingress-neg-api"},"type":"ClusterIP"}}
  creationTimestamp: "2022-11-24T04:01:41Z"
  name: ingress-neg-api
  namespace: default
  resourceVersion: "75752"
  uid: 5687946c-096f-4c77-b945-4011e86221a0
spec:
  clusterIP: 10.25.130.238
  clusterIPs:
  - 10.25.130.238
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - port: 8000
    protocol: TCP
    targetPort: 8000
  selector:
    app: ingress-neg-api
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

### 3.3 Screenshots

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


## 4. LoadBalancer Type with NodePort

### 4.1 Object Spec

NOTE: `Ingress` is not required when creating a Service with the `LoadBalancer` type.

[loadbalancer-type-api-template.yaml](app/loadbalancer-type-api-template.yaml):

| Kind    | Element                     | Value             | Description             |
|---------|-----------------------------|-------------------|-------------------------|
| Service | spec.type                   | LoadBalancer      |                         |
| Service | spec.ports.port             | 80                | You can access with `<external-endpoint-ip>:<port>`. |
| Service | spec.ports.nodePort         | 31000             | This element is **optional** and GKE will assign a port in 30000-32768 range automatically if you do not aggign it. |

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
  externalTrafficPolicy: Local
  ports:
    - port: 80
      targetPort: 8000
      nodePort: 31000
      protocol: TCP
```

### 4.2 Deploy loadbalancer-type-api

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" loadbalancer-type-api-template.yaml > loadbalancer-type-api.yaml
cat loadbalancer-type-api.yaml

kubectl apply -f loadbalancer-type-api.yaml
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=loadbalancer-type-api

kubectl describe pods loadbalancer-type-api
```

```bash
kubectl get service loadbalancer-type-api --output yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    app: loadbalancer-type-api
    cloud.google.com/neg: '{"ingress":true}'
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"app":"loadbalancer-type-api"},"name":"loadbalancer-type-api","namespace":"default"},"spec":{"externalTrafficPolicy":"Local","ports":[{"nodePort":31000,"port":80,"protocol":"TCP","targetPort":8000}],"selector":{"app":"loadbalancer-type-api"},"type":"LoadBalancer"}}
  creationTimestamp: "2022-11-24T04:10:30Z"
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
  name: loadbalancer-type-api
  namespace: default
  resourceVersion: "81655"
  uid: e2573c24-fc45-4bb7-9c89-2521a4bf5139
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.25.131.128
  clusterIPs:
  - 10.25.131.128
  externalTrafficPolicy: Local
  healthCheckNodePort: 31456
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 31000
    port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    app: loadbalancer-type-api
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 34.133.110.139
```

### 4.3 Screenshots

- Services & Ingress > SERVICES

    https://console.cloud.google.com/kubernetes/discovery

    ![loadbalancer-type](./screenshots/lbtype-services.png?raw=true)

- Services & Ingress > Service details > OVERVIEW

    ![loadbalancer-type](./screenshots/lbtype-services-overview.png?raw=true)

- Services & Ingress > Service details > DETAILS

    ![loadbalancer-type](./screenshots/lbtype-services-details.png?raw=true)

---


## 5. LoadBalancer Type with NodePort

### 5.1 Object Spec

NOTE: `Ingress` is not required when creating a Service with the `NodePort` type.

[nodeport-type-api-template.yaml](app/nodeport-type-api-template.yaml):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-type-api
  annotations:
    app: nodeport-type-api
spec:
  selector:
    app: nodeport-type-api
  type: NodePort
  ports:
    - port: 80
      targetPort: 8000
      protocol: TCP
```

### 5.2 Deploy nodeport-type-api

```bash
sed -e "s|<project-id>|${PROJECT_ID}|g" nodeport-type-api-template.yaml > nodeport-type-api.yaml
cat nodeport-type-api.yaml

kubectl apply -f nodeport-type-api.yaml
```

Confirm that pod configuration and logs after deployment:

```bash
kubectl logs -l app=nodeport-type-api

kubectl describe pods
```

```bash
kubectl get service nodeport-type-api --output yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    app: nodeport-type-api
    cloud.google.com/neg: '{"ingress":true}'
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"app":"nodeport-type-api"},"name":"nodeport-type-api","namespace":"default"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":8000}],"selector":{"app":"nodeport-type-api"},"type":"NodePort"}}
  creationTimestamp: "2022-11-24T04:24:47Z"
  name: nodeport-type-api
  namespace: default
  resourceVersion: "90492"
  uid: 33f1feab-0846-4326-8fc0-e0704194653b
spec:
  clusterIP: 10.25.129.2
  clusterIPs:
  - 10.25.129.2
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 31854
    port: 80
    protocol: TCP
    targetPort: 8000
  selector:
    app: nodeport-type-api
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

### 5.3 Screenshots

---

## 6. Cleanup

```bash
kubectl delete -f app/ingress-neg-api.yaml
kubectl delete -f app/loadbalancer-type-api.yaml

gcloud container clusters delete sample-cluster
```

## References

- Cloud Load Balancing > Documentation > Guides
    - [Serverless network endpoint groups overview](https://cloud.google.com/load-balancing/docs/negs/serverless-neg-concepts)

<br/>

- Google Kubernetes Engine (GKE) > Documentation > Guides

    - [Exposing applications using services](https://cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps)
    - [GKE Ingress for HTTP(S) Load Balancing](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress)
    - [Container-native load balancing through Ingress](https://cloud.google.com/kubernetes-engine/docs/how-to/container-native-load-balancing)
    - [Container-native load balancing through standalone zonal NEGs](https://cloud.google.com/kubernetes-engine/docs/how-to/standalone-neg)