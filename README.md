# Wazuh, Elasticsearch, and Kibana Kubernetes Deployment

This repository contains Kubernetes manifests for deploying a complete monitoring and security stack consisting of:

- Wazuh Manager (security monitoring)
- Wazuh Indexer
- Elasticsearch (2-node cluster)
- Kibana (visualization dashboard)

## Prerequisites

* Before deploying, ensure you have:
* Kubernetes cluster running on VMware vSphere
* kubectl installed and configured to access your cluster
* Sufficient resources in your vSphere environment:

  * At least 6 vCPUs and 8GB RAM available
  * At least 100GB of storage capacity
* Ingress controller installed in your cluster (e.g., NGINX Ingress)
* Domain names configured for accessing Kibana and Wazuh API

## Deployment Instructions

### 1. Clone this repository

```
git clone https://github.com/yourusername/wazuh-elk-k8s.git
cd wazuh-elk-k8s
```

### 2. Update configuration files

Before deploying, update the following:

- In `elasticsearch-secret.yaml` and `wazuh-secret.yaml`:
- Change the default passwords (base64 encode your passwords)

```shellscript
echo -n "your-secure-password" | base64
```

- In `ingress.yaml`:
- Update the hostnames to match your domain names

### 3. Create the monitoring namespace

```shellscript
kubectl apply -f namespace.yaml
```

### 4. Create the vSphere storage class

```shellscript
kubectl apply -f vsphere-storage-class.yaml
```

### 5. Create secrets

```shellscript
kubectl apply -f elasticsearch-secret.yaml
kubectl apply -f wazuh-secret.yaml
```

### 6. Deploy Elasticsearch

```shellscript
kubectl apply -f elasticsearch-statefulset.yaml
```

Wait for the Elasticsearch pods to be in Running state:

```shellscript
kubectl get pods -n monitoring -l app=elasticsearch -w
```

### 7. Deploy Wazuh components

First, create the persistent volume claims:

```shellscript
kubectl apply -f wazuh-manager-pvcs.yaml
```

Then deploy the Wazuh indexer:

```shellscript
kubectl apply -f wazuh-indexer-statefulset.yaml
```

Wait for the Wazuh indexer pod to be in Running state:

```shellscript
kubectl get pods -n monitoring -l app=wazuh-indexer -w
```

Next, create the ConfigMap for Logstash:

```shellscript
kubectl apply -f logstash-config-map.yaml
```

Finally, deploy the Wazuh manager with Logstash:

```shellscript
kubectl apply -f wazuh-manager-logstash-deployment.yaml
```

### 8. Deploy Kibana

```shellscript
kubectl apply -f kibana-deployment.yaml
```

Wait for the Kibana pod to be in Running state:

```shellscript
kubectl get pods -n monitoring -l app=kibana -w
```

### 9. Create the ingress for external access

```shellscript
kubectl apply -f ingress.yaml
```

## Post-Deployment Configuration

### Verify all components are running

```shellscript
kubectl get pods -n monitoring
```
All pods should be in the `Running` state.

### Access Kibana

1. Navigate to `https://kibana.yourdomain.com` in your browser
2. Log in with:
3. Username: `elastic`
4. Password: (the password you set in `elasticsearch-secret.yaml`)

### Configure Wazuh in Kibana

1. In Kibana, navigate to the Wazuh app (you may need to install it from the Kibana plugin menu)
2. Configure the connection to the Wazuh manager:
3. URL: `http://wazuh.yourdomain.com`
4. Port: `55000`
5. Username: `wazuh`
6. Password: (the password you set in `wazuh-secret.yaml`)

### Add Agents to Wazuh

1. In the Wazuh app in Kibana, go to "Agents" and click "Deploy new agent"
2. Follow the instructions to deploy agents on your systems

## Important Notes

The current configuration has been optimized for:
- Running without SSL (all connections are HTTP, not HTTPS)
- Using Wazuh Manager combined with Logstash in the same pod
- Using vSphere storage class for all persistent volumes
