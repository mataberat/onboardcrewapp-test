# onboardcrewapp.com test

## 1. How do you connect domain to Kubernetes cluster? Please write the process step. mydomain.com is assumption

Answer:
Kubernetes traffic routing can be done in several options. In this example, I assume that we have 2 different setups but these are depends on the Kubernetes Ingress Controller. The setup will different if we have another Ingress Controller or Cloud-Native API Gateway, etc (e.g Ambassador, Istio, etc).

#### - AWS Ingress Controller

Creating service to expose Pod inside the cluster. In this example, I will use NodePort service type. This will expose the service Port into Node worker that will be listened by the AWS Load Balancer.

##### Step 1: Creating the Kubernetes Service

```YAML
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: nginx
type: NodePort
```

##### Step 2: Creating Ingress with AWS Ingress Controller, this will spawn a new AWS Application Load Balancer

```YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healty-threshold-count: "1"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
  name: nginx
spec:
  rules:
    - host: mydomain.com
      http:
        paths:
          - backend:
              service:
                name: nginx
                port: 80
            path: /
            pathType: Prefix   
```

This will create an Ingress with exposed Application Load Balancer.

```SHELL
NAME    CLASS     HOSTS               ADDRESS                                        PORTS     AGE
nginx   aws-alb   mydomain.com   k8s-nginx-RANDOM-STRING.REGION.elb.amazonaws.com   80, 443   
```

At this point. We can add a CNAME record on DNS Management.

```SHELL
RECORD  ADDRESS           DEST                                                TTL
CNAME   mydomain.com      k8s-nginx-RANDOM-STRING.REGION.elb.amazonaws.com    300
```

#### - With no Ingress Controller

Assuming that we don't have Ingress Controller. The setup of environment is running the self-managed Load Balancer outside the cluster. The example will use Nginx as basic Load Balancer with reverse proxy configuration.

##### Step 1: Configuring the Nginx Load Balancer

```SHELL
http {
    upstream backend {
        server kubernetes.cluster.node-1.worker:80;
        server kubernetes.cluster.node-2.worker:80;
    }
    
    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```

##### Step 2: Create Kubernetes service with NodePort

```YAML
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: nginx
type: NodePort
```

At this point. We can add a A record on DNS Management.

```SHELL
RECORD  ADDRESS           DEST                                                TTL
A       mydomain.com      IP-of-Nginx-Reverse-proxy                           300
```

## 2. Modify below docker-compose.yaml to map given react application to nginx from running 3000 to nginx 80

Before

```YAML
version: 3.8
services:
  web:
    build:
      context: ./
      target: runner
    volumes:
      - ./:/app
    command: npm run dev
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
```

After

```YAML
version: 3.8
services:
  web:
    image: nginx:latest
  ports:
    - "80:80" # Default export port of nginx is 80, reference https://hub.docker.com/_/nginx
  volumes:
    - /static_assets/:/usr/share/nginx/html
```

## 3. Assuming you have wildcard certificate. How do you insert certificate to the Kubernetes cluster?

Answer: Kubernetes Secret supports the type of TLS certificate, and Kubernetes Ingress supports SSL Termination as well. These are the steps to import the wildcard certs and use it to perform SSL Termination with Ingress.

##### Step 1: Creating a Secrets

Kubernetes secrets only support for base64 format.

```YAML
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: wildcard-mydomain-secret
data:
  tls.crt: "<base64 encoded cert>"
  tls.key: "<base64 encoded private key>"
  ca.crt: "<base64 encoded intermediate certificate>"
```

Or you can do this step to have the same functionality.

```SHELL
kubectl create secret generic wildcard-mydomain-secret --from-file=tls.crt=mydomain.com.crt --from-file=tls.key=mydomain.com.private.key --from-file=ca.crt=comodo-ssl.ca.crt
```

The imported certs should be like this.

```SHELL
-----BEGIN CERTIFICATE-----
mydomain.com.crt
-----END CERTIFICATE-----

-----BEGIN RSA PRIVATE KEY-----
mydomain.com.private.key
-----END RSA PRIVATE KEY-----

-----BEGIN CERTIFICATE-----
comodo-ssl.ca.crt
-----END CERTIFICATE-----
```

##### Step 1: Use the certificate for SSL termination on Ingress Nginx Controller

This will create 2 Ingress with two different exposed endpoint app.mydomain.com and api.mydomain.com.

```YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
    - host: api.mydomain.com
      http:
        paths:
          - backend:
              service:
                name: api-backend
                port: 80
            path: /core/v1/app
            pathType: Prefix  
  tls:
    - hosts:
        - api.mydomain.com
      secretName: wildcard-mydomain-secret
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
    - host: app.mydomain.com
      http:
        paths:
          - backend:
              service:
                name: app-frontend
                port: 80
            path: /
            pathType: Prefix  
  tls:
    - hosts:
        - app.mydomain.com
      secretName: wildcard-mydomain-secret
```
