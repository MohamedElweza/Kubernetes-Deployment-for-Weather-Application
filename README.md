# Project Documentation: Kubernetes Deployment for Weather Application

## Overview

This document outlines the complete steps followed to containerize and deploy a multi-service weather application using Kubernetes. It covers Dockerization, creation of Kubernetes manifests, Secrets, ConfigMaps, StatefulSets, Services, Ingress with TLS, and initialization jobs for MySQL.

## Project Structure

```
.
├── authentication
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── statefulset.yaml
│   ├── headless-service.yaml
│   ├── mysql-init-job.yaml
│   └── secret.yaml
├── ui
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── tls.crt
│   └── tls.key
├── weather
│   ├── deployment.yaml
│   ├── service.yaml
│   └── secret.yaml
└── Dockerfiles (UI & Weather app in Go)
```

---

## Step-by-Step Guide

### 1. Dockerization

* **UI**: Built using a frontend framework (e.g., React/Vue).

  * Created a Dockerfile to build and serve the frontend using NGINX.
* **Weather**: A Go application.

  * Created a Dockerfile to compile and serve the Go application.

### 2. MySQL Setup

* Used a **StatefulSet** to deploy MySQL with persistence.
* Created **Secrets** to store the root and user passwords.

```bash
kubectl create secret generic mysql-secret \
  --from-literal=root-password=yourRootPass \
  --from-literal=auth-password=yourUserPass
```

* Created a **headless service** for internal communication:

```yaml
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
```

* Created a **Job** to initialize the database and user:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mysql-init-job
spec:
  template:
    spec:
      containers:
      - name: mysql-init-container
        image: mysql:5.7
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        - name: MYSQL_AUTH_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: auth-password
        - name: MYSQL_HOST
          value: "mysql"
        - name: MYSQL_PORT
          value: "3306"
        command: ["/bin/sh", "-c"]
        args:
        - |
          mysql -h${MYSQL_HOST} -P${MYSQL_PORT} -uroot -p${MYSQL_ROOT_PASSWORD} <<EOF
          CREATE DATABASE IF NOT EXISTS weatherapp;
          CREATE USER '${MYSQL_USER}'@'%' IDENTIFIED BY '${MYSQL_AUTH_PASSWORD}';
          GRANT ALL PRIVILEGES ON weatherapp.* TO '${MYSQL_USER}'@'%';
          FLUSH PRIVILEGES;
          EOF
      restartPolicy: Never
```

### 3. Authentication Service

* Uses environment variables (via Secrets) to connect to MySQL.
* Has its own **Deployment** and **Service** files.

### 4. UI Service

* Deployed as a Deployment.
* Exposed using a Kubernetes **Ingress** with TLS.

```bash
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

* The Ingress routes traffic to the UI based on path or host.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ui-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - yourdomain.com
    secretName: tls-secret
  rules:
  - host: yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ui
            port:
              number: 80
```

### 5. Weather Service

* Written in Go and dockerized.
* Uses **Deployment**, **Service**, and **Secrets** to connect to the MySQL DB.

---

## Deployment Order

To ensure everything runs smoothly, deploy in the following order:

1. **Secrets**: `mysql-secret`, app secrets.
2. **MySQL StatefulSet + Headless Service**.
3. **MySQL Init Job**.
4. **Weather Service** (Deployment + Service + Secret).
5. **Authentication Service** (StatefulSet/Deployment + Service).
6. **UI Service** (Deployment + Service + Ingress).
7. **TLS Secret**.

---

## Conclusion

This deployment showcases a full-stack Kubernetes application with proper service separation, secret management, persistent state handling via StatefulSets, and secure TLS access using Ingress. Each component is isolated, scalable, and easily maintainable within a production-grade Kubernetes cluster.

