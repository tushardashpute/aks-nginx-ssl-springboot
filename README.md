# Deploy NGINX Ingress on AKS with SSL and Spring Boot App

This guide walks you through deploying NGINX Ingress Controller on Azure Kubernetes Service (AKS), running a Spring Boot application on port `33333`, and configuring a self-signed SSL certificate for HTTPS access and registering it in Chrome browser.

---

## ✅ Prerequisites
- Azure AKS cluster created
- `kubectl` configured for your AKS cluster
- `helm` installed
- `openssl` installed

---

## 1️⃣ Install NGINX Ingress Controller on AKS
```bash
kubectl create namespace ingress-nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.publishService.enabled=true
```

---

## 2️⃣ Create Self-Signed SSL Certificate

### Create OpenSSL config with SAN:
Create a file `openssl-san.cnf`:
```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[dn]
C=US
ST=Some-State
L=.
O=spring
OU=it
CN=spring.astute001.com
emailAddress=tdashpute@gmail.com

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = spring.astute001.com
```

### Generate Key, CSR and Certificate:
```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -config openssl-san.cnf
openssl x509 -req -in server.csr -signkey server.key -out server.crt \
  -days 365 -extensions req_ext -extfile openssl-san.cnf
```

### Create TLS Secret in Kubernetes:
```bash
kubectl create secret tls tls-secret \
  --cert=server.crt \
  --key=server.key \
  --namespace default
```

---

## 3️⃣ Deploy Spring Boot App on Port 33333

### springboot-deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springboot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot
  template:
    metadata:
      labels:
        app: springboot
    spec:
      containers:
      - name: springboot
        image: tushardashpute/springboot:latest
        ports:
        - containerPort: 33333
---
apiVersion: v1
kind: Service
metadata:
  name: springboot
spec:
  selector:
    app: springboot
  ports:
  - port: 80
    targetPort: 33333
```

Apply:
```bash
kubectl apply -f springboot-deployment.yaml
```

---

## 4️⃣ Create Ingress Resource with SSL

### springboot-ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: springboot-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - spring.astute001.com
    secretName: tls-secret
  rules:
  - host: spring.astute001.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: springboot
            port:
              number: 80
```

Apply:
```bash
kubectl apply -f springboot-ingress.yaml
```

---

## 5️⃣ Update Hosts File (Local DNS Resolution)
Edit `/etc/hosts` or `C:\Windows\System32\drivers\etc\hosts`:
```
<INGRESS_EXTERNAL_IP> spring.astute001.com
```

To get the external IP:
```bash
kubectl get svc -n ingress-nginx
```

---

## 6️⃣ Register Self-Signed Cert in Chrome (via OS Trust Store)

### On Linux:
```bash
sudo cp server.crt /usr/local/share/ca-certificates/spring.astute001.com.crt
sudo update-ca-certificates
```

### On macOS:
- Open **Keychain Access**
- Import `server.crt`
- Set **"Always Trust"** under certificate properties

### On Windows:
- Run `certmgr.msc`
- Import `server.crt` into **Trusted Root Certification Authorities**

---

## ✅ Done!
Now you can access your Spring Boot app at:
```
https://spring.astute001.com
```
And Chrome will trust your self-signed certificate!

