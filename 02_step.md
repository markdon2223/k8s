# **Documentation: Deploying CVAT on Azure Kubernetes Service (AKS)**

## **Overview**

This document provides a step-by-step guide for deploying CVAT (Computer Vision Annotation Tool) on Azure Kubernetes Service (AKS). The guide covers setting up the AKS cluster, deploying CVAT and its dependencies, configuring Traefik as the ingress controller, and making CVAT accessible via a domain name.

## **Table of Contents**

1. [Prerequisites](#prerequisites)
2. [Set Up Azure Kubernetes Service (AKS) Cluster](#set-up-azure-kubernetes-service-aks-cluster)
3. [Deploy CVAT and Dependencies](#deploy-cvat-and-dependencies)
4. [Configure Traefik Ingress Controller](#configure-traefik-ingress-controller)
5. [Expose CVAT Service](#expose-cvat-service)
6. [Configure DNS Records](#configure-dns-records)
7. [(Optional) Set Up HTTPS with Traefik](#optional-set-up-https-with-traefik)
8. [Verify the Deployment](#verify-the-deployment)
9. [Troubleshooting](#troubleshooting)
10. [References](#references)

## **1. Prerequisites**

Before starting the deployment, ensure you have the following:

- **Azure Account:** An active Azure subscription.
- **Azure CLI:** Installed and configured ([Download & Install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)).
- **kubectl CLI:** Installed and configured ([Download & Install](https://kubernetes.io/docs/tasks/tools/install-kubectl/)).
- **Helm CLI:** Installed and configured ([Download & Install](https://helm.sh/docs/intro/install/)).
- **Domain Name:** A domain name for accessing CVAT (optional for HTTP, required for HTTPS).

## **2. Set Up Azure Kubernetes Service (AKS) Cluster**

Follow these steps to create an AKS cluster:

### **2.1. Log in to Azure**

```sh
az login
```

### **2.2. Create a Resource Group**

Replace `myResourceGroup` with your desired resource group name and `eastus` with your preferred region.

```sh
az group create --name myResourceGroup --location eastus
```

### **2.3. Create the AKS Cluster**

Replace `myAKSCluster` with your desired AKS cluster name.

```sh
az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 3 --enable-addons monitoring --generate-ssh-keys
```

### **2.4. Connect to the AKS Cluster**

```sh
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```

## **3. Deploy CVAT and Dependencies**

Download the CVAT Helm chart and configure the required values for the deployment.

### **3.1. Add the Helm Repository**

```sh
helm repo add cvat https://cvat.github.io/cvat-charts/
helm repo update
```

### **3.2. Create a Namespace for CVAT**

```sh
kubectl create namespace cvat
```

### **3.3. Deploy CVAT Using Helm**

Create a `values.yaml` file for custom configurations:

```yaml
# values.yaml

# Backend Service
backend:
  image:
    repository: cvat/cvat
    tag: 1.8.0 # Use the desired version

# Frontend Service
frontend:
  image:
    repository: cvat/cvat
    tag: 1.8.0 # Use the desired version

# PostgreSQL Database
postgresql:
  image:
    repository: postgres
    tag: 13

# Redis
redis:
  image:
    repository: redis
    tag: 7

# ClickHouse
clickhouse:
  image:
    repository: yandex/clickhouse-server
    tag: 22.8.10.7

# Kvrocks
kvrocks:
  image:
    repository: kvrocks/kvrocks
    tag: 2.2.0

# Traefik
traefik:
  enabled: true
  ingressClassName: cvat-traefik
```

Install CVAT with Helm:

```sh
helm install cvat cvat/cvat -f values.yaml -n cvat
```

## **4. Configure Traefik Ingress Controller**

Traefik will manage the ingress traffic to CVAT.

### **4.1. Verify Traefik Deployment**

Ensure Traefik is deployed and has an external IP.

```sh
kubectl get services -n cvat
```

**Check for Traefik's `LoadBalancer` service**:

```plaintext
NAME           TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                          AGE
cvat-traefik   LoadBalancer   10.0.188.143   40.78.111.210   80:31415/TCP,443:32668/TCP                      55m
```

### **4.2. Create an Ingress Resource**

Create an `Ingress` configuration file:

```yaml
# cvat-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cvat-ingress
  namespace: cvat
  annotations:
    traefik.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: cvat-traefik
  rules:
  - host: cvat.yourdomain.com  # Replace with your domain or subdomain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cvat1-frontend-service
            port:
              number: 80
```

Apply the `Ingress` configuration:

```sh
kubectl apply -f cvat-ingress.yaml
```

## **5. Expose CVAT Service**

### **5.1. Configure DNS Records**

Update your DNS settings to point your domain to the Traefik `LoadBalancer` IP.

1. **Log in to your Domain Registrar's DNS Management Console.**
2. **Create an A Record**:
   - **Name:** `cvat` (or `www` for `www.cvat.yourdomain.com`)
   - **Type:** `A`
   - **Value:** `40.78.111.210` (Traefik LoadBalancer IP)
   - **TTL:** `3600` or default

**Example DNS Record:**

```plaintext
cvat       A       40.78.111.210
```

### **5.2. Check DNS Propagation**

Ensure the DNS records are propagated by running:

```sh
nslookup cvat.yourdomain.com
```

## **6. (Optional) Set Up HTTPS with Traefik**

For HTTPS, set up TLS certificates using Traefik and Let's Encrypt.

### **6.1. Create an Ingress Resource for TLS**

Create `cvat-ingress-tls.yaml`:

```yaml
# cvat-ingress-tls.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cvat-ingress
  namespace: cvat
  annotations:
    traefik.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: cvat-traefik
  rules:
  - host: cvat.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cvat1-frontend-service
            port:
              number: 80
  tls:
  - hosts:
    - cvat.yourdomain.com
    secretName: cvat-tls
```

Apply the TLS configuration:

```sh
kubectl apply -f cvat-ingress-tls.yaml
```

### **6.2. Install `cert-manager`**

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager-v1.12.0.yaml
```

### **6.3. Create the `ClusterIssuer` for Let's Encrypt**

Create `letsencrypt-clusterissuer.yaml`:

```yaml
# letsencrypt-clusterissuer.yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com  # Replace with your email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: cvat-traefik
```

Apply the `ClusterIssuer` configuration:

```sh
kubectl apply -f letsencrypt-clusterissuer.yaml
```

## **7. Verify the Deployment**

### **7.1. Check CVAT Access**

Visit:

- **HTTP:** `http://cvat.yourdomain.com`
- **HTTPS:** `https://cvat.yourdomain.com` (if HTTPS is configured)

### **7.2. Verify TLS Certificate**

Check if the TLS certificate is issued correctly.

```sh
kubectl describe certificate cvat-tls -n cvat
```

## **8. Troubleshooting**

Here are some common issues and solutions:

### **8.1. DNS Issues**

-

 **Issue:** Domain not resolving.
- **Solution:** Ensure DNS records are correct and propagated. Use `nslookup` to verify.

### **8.2. Network Security Group Issues**

- **Issue:** Unable to access CVAT.
- **Solution:** Ensure NSG rules allow inbound traffic on ports 80 and 443.

### **8.3. Traefik Logs**

- **Issue:** Traefik errors.
- **Solution:** Check Traefik logs for errors related to routing or certificates.

```sh
kubectl logs -n cvat deploy/cvat1-traefik
```

## **9. References**

- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)

## **10. Contact**

For additional support or questions, you can reach out to:

- **Tanvir:** [Your Email]
- **AKS Support:** [Azure Support](https://azure.microsoft.com/en-us/support/options/)

---

This documentation covers the end-to-end process of deploying CVAT on AKS, configuring Traefik, and making CVAT accessible via your domain. If you need further customization or face specific issues, you can adapt the steps or seek additional support.
