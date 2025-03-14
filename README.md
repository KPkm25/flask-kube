
#  **Deploying a Flask Application on Kubernetes with Auto-Scaling & Load Testing**

## **Overview**
This guide walks through deploying a **Flask application** on **Kubernetes**, handling **authentication issues**, enabling **auto-scaling (HPA)**, and performing **load testing**.

---

## **Prerequisites**
Ensure you have the following installed:
- **Docker** (for building images)
- **Kubernetes cluster** (with `master-vm`, `worker1-vm`, `worker2-vm`)
- **Container runtime** (`containerd`)
- **Metrics Server** (for auto-scaling)

---

## **1️⃣ Building & Containerizing the Flask Application**

### **Flask Application (`app.py`)**
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify(message="Hello, World! This is a Flask app running in Docker.")

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
> **Issue:** Flask bound to `127.0.0.1` won't be accessible.
> **Fix:** Use `app.run(host="0.0.0.0", port=5000)`.

### **Dockerfile**
```dockerfile
FROM python:3.11
WORKDIR /app
COPY . /app
RUN pip install flask
EXPOSE 5000
CMD ["python", "app.py"]
```
> **Issue:** Missing `EXPOSE 5000` prevents service access.
> **Fix:** Ensure `EXPOSE 5000` is included.

### **Build & Push Image**
```bash
docker build -t kpkm25/flask-kube .
docker push kpkm25/flask-kube
```
> **Issue:** `Permission denied while connecting to Docker daemon`
> **Fix:** `sudo usermod -aG docker $USER && newgrp docker`

---

## **2️⃣ Deploying Flask App on Kubernetes**

### **Deployment & Service YAML (`deployment-service.yaml`)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
        - name: flask-container
          image: kpkm25/flask-kube:latest
          ports:
            - containerPort: 5000
          resources:
            requests:
              cpu: "100m"
            limits:
              cpu: "250m"
      imagePullSecrets:
        - name: docker-secret
---
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: NodePort
```
> **Issue:** `ErrImagePull` due to unauthenticated Docker pulls.
> **Fix:** Authenticate Kubernetes with Docker Hub.

### **Apply Deployment**
```bash
kubectl apply -f deployment-service.yaml
```

---

## **3️⃣ Fixing Docker Hub Rate Limits (Authentication Issue)**
> **Issue:**
```
Failed to pull image "curlimages/curl": toomanyrequests: You have reached your unauthenticated pull rate limit.
```
> **Fix:** Authenticate Kubernetes with Docker Hub.

### **Solution: Create Docker Secret**
```bash
kubectl create secret docker-registry docker-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=kpkm25 \
  --docker-password=YOUR_DOCKER_HUB_PASSWORD \
  --docker-email=YOUR_EMAIL
```

### **Patch Default Service Account**
```bash
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "docker-secret"}]}'
```
> **Fix applied!** Now Kubernetes will authenticate with Docker Hub and avoid rate limits.

---

## **4️⃣ Installing & Troubleshooting Metrics Server**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
> **Issue:** `x509: certificate signed by unknown authority`
> **Fix:** Edit `metrics-server` deployment and add:
```yaml
- --kubelet-insecure-tls
```
```bash
kubectl rollout restart deployment -n kube-system metrics-server
```

---

## **5️⃣ Enabling HPA (Horizontal Pod Autoscaler)**
```bash
kubectl autoscale deployment flask-app --cpu-percent=50 --min=3 --max=10
kubectl get hpa
```
> **Issue:** `Metrics API not available`
> **Fix:** Restart Metrics Server.

---

## **6️⃣ Load Testing & Debugging NodePort Issues**

### **Testing Service Internally**
```bash
kubectl run -it --rm busybox --image=busybox -- /bin/sh
wget -q -O- http://10.97.210.48:80
```

### **Finding NodePort & Testing External Access**
```bash
kubectl get svc flask-service
```
Example Output:
```
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
flask-service   NodePort   10.100.195.164   <none>        80:30455/TCP   107s
```
```bash
curl http://192.168.147.129:30455
```
> **Issue:** `Connection refused`
> **Fix:** Restart `kube-proxy`:
```bash
kubectl rollout restart daemonset kube-proxy -n kube-system
```

---

## **7️⃣ Simulating Load for HPA**
```bash
kubectl run -it --rm load-generator --image=busybox -- /bin/sh
while true; do wget -q -O- http://192.168.147.129:30455; done
```

### **Check Scaling**
```bash
kubectl get hpa
kubectl get pods
```



