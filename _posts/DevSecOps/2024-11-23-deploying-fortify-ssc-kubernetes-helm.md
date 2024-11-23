---
title: "Deploying Fortify SSC on Kubernetes Using Helm"
classes: wide
header:
  teaser: /assets/images/DevSecOps/SSC-Kubernetes/GoJOO.jpg
ribbon: blue
description: "Step-by-step guide for deploying Fortify SSC on Kubernetes using Helm."
categories:
  - DevSecOps
toc: true
---

# Deploying Fortify SSC on Kubernetes Using Helm

This guide provides a step-by-step approach to deploy Fortify SSC on Kubernetes using Helm.

---

## **1. Prepare the Environment**

### **Create Docker Secret**
Ensure Kubernetes can pull images from private Docker registries:

```bash
kubectl create secret generic regcred \
  --from-file=.dockerconfigjson=.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

### **Open Firewall Ports**
Allow necessary ports for Fortify SSC and SQL Server communication:

```bash
sudo firewall-cmd --zone=public --add-port=8443/tcp --permanent
sudo firewall-cmd --add-port=1433/tcp --permanent
sudo firewall-cmd --reload
```

---

## **2. Set Up the SQL Server Database**

### **Pull SQL Server Image**
Download the latest SQL Server image:

```bash
docker pull mcr.microsoft.com/mssql/server:2022-latest
```

### **Run SQL Server Container**
Start the SQL Server container:

```bash
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=P@ssw0rdH' \
  -p 1433:1433 --name mssql-server -d mcr.microsoft.com/mssql/server:2022-latest
```

### **Connect to SQL Server**
Verify connectivity:

```bash
docker exec -it mssql-server /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P 'P@ssw0rdH'
```

---

## **3. Prepare Fortify SSC Secrets**

### **Create Secrets Directory**
Navigate to the Helm chart directory:

```bash
cd Fortify/helm
mkdir secrets
```

### **Add Required Files**
Add these files into the `secrets/` directory:
- `fortify.license`
- `ssc.autoconfig.yaml`
- `ssc-kube.crt`
- `ssc-kube.jks`
- `ssc-kube-pass`

### **Create Kubernetes Secret**
Package the secrets:

```bash
kubectl create secret generic sscsecrets --from-file=secrets/
```

Verify the secret:

```bash
kubectl describe secret sscsecrets
```

---

## **4. Pull Fortify SSC Docker Image**

### **Log In to Docker**
Authenticate with Docker Hub:

```bash
docker login -u mohamedhesham94 -p dodoo_12FK
```

### **Pull the SSC Image**
Download the SSC image:

```bash
docker pull fortifydocker/ssc-webapp:24.2.0.0186
```

### **Create Docker Registry Secret**
Create a Kubernetes secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=mohamedhesham94 \
  --docker-password=dodoo_12FK
```

---

## **5. Deploy Fortify SSC Using Helm**

### **Install the Helm Chart**
Run the Helm command to install SSC:

```bash
helm install ssc ssc-1.1.2420186+24.2.0.0186.tgz -f ssc-values-example.yaml
```

---

## **6. Verify the Deployment**

### **Check Kubernetes Resources**
Ensure SSC is deployed:

```bash
kubectl get all
```

### **Inspect SSC Pod**
Check for issues:

```bash
kubectl describe pod ssc-webapp-0
```

### **View Logs**
Check logs:

```bash
kubectl logs statefulset.apps/ssc-webapp
```

---

## **7. Access Fortify SSC**

Access SSC via the external IP or ingress URL:

- **Default Ports:** `80` (HTTP) or `443` (HTTPS)
- Example:
  - HTTP: `http://<external-ip>`
  - HTTPS: `https://<external-ip>`

### **Local Access**
Use `kubectl port-forward` for local access:

```bash
kubectl port-forward ssc-webapp-0 8443:8443
```

URL: [https://localhost:8443](https://localhost:8443)

---

### **Troubleshooting**
- If the service does not start:
  - Check Kubernetes events: `kubectl describe service ssc-service`
  - Verify Docker image availability.
- If `LoadBalancer` is `Pending`, configure MetalLB or use `NodePort`.
