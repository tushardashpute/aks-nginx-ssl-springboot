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
Sample Output:
```bash
$ kubectl get pods,svc,ingress,ep
NAME                              READY   STATUS    RESTARTS   AGE
pod/springboot-786d744fc5-95bsl   1/1     Running   0          6h3m

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.0.0.1       <none>        443/TCP   7h36m
service/springboot   ClusterIP   10.0.195.196   <none>        80/TCP    7h31m

NAME                                           CLASS   HOSTS                  ADDRESS         PORTS     AGE
ingress.networking.k8s.io/springboot-ingress   nginx   spring.astute001.com   74.179.192.68   80, 443   7h27m

NAME                   ENDPOINTS            AGE
endpoints/kubernetes   104.45.179.218:443   7h36m
endpoints/springboot   10.244.0.133:33333   7h31m
```
---

## 5️⃣ Update Hosts File (Local DNS Resolution)
Edit `/etc/hosts` or `C:\Windows\System32\drivers\etc\hosts`:
```
<INGRESS_EXTERNAL_IP> spring.astute001.com

In my case, as I have domain registered, I created a A record for the INGRESS_EXTERNAL_IP:

![image](https://github.com/user-attachments/assets/7b4e5687-a520-458a-8da2-a0570e2fc269)

```

To get the external IP:
```bash
kubectl get svc -n ingress-nginx
```
If we access it directly now, we can access it over https but with not-secure.

![image](https://github.com/user-attachments/assets/2b694f68-8143-4df8-8584-e0f1b208aefe)


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

Now if we try to access it, it will show connection as secued with the certificate which we have created.

![image](https://github.com/user-attachments/assets/67d50c61-ded5-42aa-a1d8-885b18fb19d8)

