# MongoDB on a Managed Kubernetes Cluster (DigitalOcean)

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![DigitalOcean](https://img.shields.io/badge/DigitalOcean-0080FF?style=flat&logo=digitalocean&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat&logo=mongodb&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=flat&logo=nginx&logoColor=white)

A hands-on DevOps project that deploys a **replicated, persistent MongoDB** on DigitalOcean Kubernetes (DOKS) with a Helm chart. **Mongo Express** gives me a web UI for the database, and an **Nginx ingress controller** exposes it on my own domain.

Repository: [devops-k8s-helm-mongodb-digitalocean](https://gitlab.com/cloudbuildops/devops-k8s-helm-mongodb-digitalocean.git)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Guide](#step-by-step-guide)
  - [1. Create the cluster](#1-create-the-cluster)
  - [2. Connect to the cluster](#2-connect-to-the-cluster)
  - [3. Deploy MongoDB with Helm](#3-deploy-mongodb-with-helm)
  - [4. Deploy Mongo Express](#4-deploy-mongo-express)
  - [5. Deploy the Nginx ingress controller](#5-deploy-the-nginx-ingress-controller)
  - [6. Create the Ingress rule](#6-create-the-ingress-rule)
  - [7. Verify](#7-verify)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)
- [Author](#author)

---

## Overview

Running a database on Kubernetes needs more than a Deployment: stable pod identities, persistent storage and a safe way to hand out credentials. Writing all of that by hand is a lot of YAML, so I use a Helm chart for the database and keep my own manifests for the parts that are specific to this setup.

What this project covers:

- Creating a managed Kubernetes cluster on DigitalOcean
- Connecting with `kubectl` through the downloaded kubeconfig
- Deploying MongoDB as a **3-member replica set (StatefulSet)** with the Bitnami chart
- Persistent storage through DigitalOcean Block Storage volumes
- Deploying Mongo Express (Deployment + Service) and wiring it to the database
- Installing the Nginx ingress controller with Helm, which provisions a DigitalOcean LoadBalancer automatically
- Routing a custom domain to Mongo Express with an Ingress rule

## Architecture

```
Browser
   |
mongo-express.dorjee.dev  (DNS -> LoadBalancer IP)
   |
DigitalOcean LoadBalancer  (created by the ingress controller)
   |
Nginx Ingress Controller
   |
mongo-express-service :8081
   |
mongo-express pod
   |
mongodb-0.mongodb-headless :27017
   |
MongoDB StatefulSet (3 replicas, each with a Block Storage volume)
```

## Prerequisites

- A **DigitalOcean account**
- **kubectl** (`brew install kubectl` on macOS)
- **Helm 3** (`brew install helm` on macOS)
- A **domain** you can point at the load balancer IP

## Project Structure

```
.
├── README.md
├── helm-mongodb.yaml       # Helm values override for the Bitnami MongoDB chart
├── helm-mongo-express.yaml      # Deployment + Service for Mongo Express
├── helm-ingress.yaml            # Ingress rule for the custom domain
└── screenshots/            # Images used in this README
```

> Keep your kubeconfig (`test-kubeconfig.yaml`) **out of the repository**. Add it to `.gitignore`.

## Step-by-Step Guide

### 1. Create the cluster

In the DigitalOcean console go to **Core Cloud → Kubernetes → Create a Kubernetes Cluster** and choose:

| Setting | Value |
|---|---|
| Kubernetes version | Latest available (recommended) |
| Datacenter region | Frankfurt (FRA1), the closest one to me |
| Node pool type | CPU |
| Nodes | 2 |
| vCPU / RAM / Storage per node | 2 shared vCPU / 4 GiB / 80 GiB |
| High availability (control plane) | Disabled |

<img src="screenshots/create-kubernetes-cluster.png" alt="digital-ocean-kubernetes-cluster" width="600">

I turned HA off on purpose. It adds a redundant control plane for an extra monthly fee, which only makes sense for production workloads. For a demo cluster it's unnecessary cost, and the worker nodes still run my workloads normally. For anything production-grade I would enable it.

### 2. Connect to the cluster

Download the kubeconfig file from the cluster page, restrict its permissions and point `kubectl` at it:

```bash
chmod 400 test-kubeconfig.yaml
export KUBECONFIG=test-kubeconfig.yaml
kubectl get nodes
```

Both nodes should show up as `Ready`, so the connection works.

**Screenshot:** `kubectl get nodes`

<img src="screenshots/digital-ocean-nodes.png" alt="kubectl-get-nodes" width="600">

### 3. Deploy MongoDB with Helm

There are two ways to deploy a StatefulSet like MongoDB:

1. Write all the config files yourself (StatefulSet, services, secrets, volumes, ...)
2. Use a bundle of those files, a Helm chart

I went with option 2. The Bitnami chart already handles the replica set setup, the headless service, the secret and the persistent volume claims.

Add the repo and look around:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo bitnami
helm search repo bitnami/mongodb
```

The chart parameters are documented on its Artifact Hub page, so I only override what I need in `helm-mongodb.yaml`:

```yaml
architecture: replicaset
replicaCount: 3
persistence:
  storageClass: "do-block-storage"
auth:
  rootPassword: <ROOT_PASSWORD>
```

Install it:

```bash
helm install mongodb --values helm-mongodb.yaml bitnami/mongodb
kubectl get all
kubectl get secret
```

The secret `mongodb` contains the root password I provided. DigitalOcean also creates Block Storage volumes for the MongoDB pods through the `do-block-storage` storage class. They show up under **Volumes** in the console.

> **Security note:** don't commit real credentials to Git. Pass the password with `--set auth.rootPassword=...`, reference an existing secret, or use a secrets manager.

**Screenshot:** DigitalOcean Volumes page


<img src="screenshots/volumes-block-storage.png" alt="digitalocean-volumes" width="600">

### 4. Deploy Mongo Express

Mongo Express is a single pod with a single service. The admin password is read from the secret the Helm chart created, so nothing sensitive sits in the manifest.

`helm-mongo-express.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          value: root
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb
              key: mongodb-root-password
        - name: ME_CONFIG_MONGODB_URL
          value: "mongodb://$(ME_CONFIG_MONGODB_ADMINUSERNAME):$(ME_CONFIG_MONGODB_ADMINPASSWORD)@mongodb-0.mongodb-headless:27017"
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
```

```bash
kubectl apply -f helm-mongo-express.yaml
```
**Screenshot:** helm mongo express pods


<img src="screenshots/apply-helm-mongo-express.png" alt="mongo express pods" width="600">


**Why `mongodb-0.mongodb-headless`?** A headless service lets you talk to one specific pod directly instead of being load balanced across all of them. In a StatefulSet, pods get stable names with an index, and index `0` is the first pod. This address tells Mongo Express to connect to the first MongoDB member.

### 5. Deploy the Nginx ingress controller

I installed the controller from its Helm chart:

```bash
helm install nginx-ingress oci://ghcr.io/nginx/charts/nginx-ingress \
  --set controller.reportIngressStatus.enable=true
```

This also created a DigitalOcean LoadBalancer automatically. The load balancer is the entry point to the cluster, so all outside traffic comes in through its IP.

```bash
kubectl get svc
```

**Screenshot:** LoadBalancer in the DigitalOcean console

<img src="screenshots/load-balancer-digital-ocean.png" alt="digitalocean-loadbalancer" width="600">

### 6. Create the Ingress rule

First I pointed a DNS record for my domain (`mongo-express.dorjee.dev`) to the load balancer IP. Then I created the rule that sends the traffic to Mongo Express.

`helm-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mongo-express
spec:
  ingressClassName: nginx
  rules:
  - host: mongo-express.dorjee.dev   # domain pointing to the LoadBalancer IP
    http:
      paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: mongo-express-service
              port:
                number: 8081
```

```bash
kubectl apply -f helm-ingress.yaml
```

**Screenshot:** `helm ingress`

<img src="screenshots/helm-ingress.png" alt="helm ingress" width="600">



After DNS propagates, Mongo Express is reachable at `http://mongo-express.dorjee.dev`.

### 7. Verify

```bash
kubectl get pods
kubectl get pvc
kubectl get svc
kubectl get ingress
```



<img src="screenshots/mongo-express-ui.png" alt="mongo-express-ui" width="600">

## Cleanup

DigitalOcean bills for nodes, volumes and load balancers, so I tear everything down when I'm done:

```bash
kubectl delete -f helm-ingress.yaml
kubectl delete -f helm-mongo-express.yaml
helm uninstall nginx-ingress
helm uninstall mongodb
kubectl delete pvc --all
```

Then delete the cluster in the DigitalOcean console and check that no leftover volumes or load balancers remain.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Mongo Express pod in `CrashLoopBackOff` | MongoDB not ready yet, or wrong credentials | Check `kubectl logs <pod>` and confirm the `mongodb` secret exists |
| MongoDB pods stuck in `Pending` | Volume could not be provisioned | Run `kubectl describe pvc` and check the storage class name `do-block-storage` |
| Domain does not open | DNS not propagated or pointing to the wrong IP | Compare the DNS record with the LoadBalancer IP from `kubectl get svc` |
| `404` from Nginx | Ingress host or service name does not match | Check `kubectl describe ingress mongo-express` |
| LoadBalancer stays in `<pending>` | Still being created by DigitalOcean | Wait a few minutes and check the console |

## Key Takeaways

- **StatefulSets vs Deployments:** databases need stable pod identities and their own storage.
- **Helm** replaces dozens of hand-written manifests with a chart plus a small values file.
- **Headless services** give direct access to individual pods.
- An **ingress controller**, a cloud **load balancer** and **DNS** work together to expose a service on a custom domain.
- Credentials belong in **Kubernetes secrets**, not in manifests or Git.
- Possible improvements: pin the image versions, add HTTPS with cert-manager, and connect Mongo Express with a replica set connection string instead of a single pod.

## Author

**N.Dorjee**
*CloudBuildOps — DevOps portfolio project*

[![GitHub](https://img.shields.io/badge/GitHub-dor--jee-181717?logo=github&logoColor=white)](https://github.com/dor-jee)
[![GitLab](https://img.shields.io/badge/GitLab-CloudBuildOps-FC6D26?logo=gitlab&logoColor=white)](https://gitlab.com/cloudbuildops/)
[![Website](https://img.shields.io/badge/Website-dorjee.dev-0A66C2?logo=googlechrome&logoColor=white)](https://dorjee.dev)
