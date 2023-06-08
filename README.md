# onboardcrewapp.com test

## 1. How do you connect domain to Kubernetes cluster? Please write the process step. mydomain.com is assumption

Answer:
Kubernetes traffic routing can be done in several options. In this example, I assume that we have 2 different setups but these are depends on the Kubernetes Ingress Controller. The setup will different if we have another Ingress Controller of Cloud-Native API Gateway, etc (e.g Ambassador, Istio, etc).

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

##### Step 2: Creating Ingress with AWS Ingress Controller, this will spawn a new AWS Application Load Balancer.**

Without TLS Termination (HTTPS Termination on AWS Load Balancer)

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

Adding TLS Termination on Ingress (add to the bottom of YAML above)

```YAML
tls:
  - hosts:
      - "*.example.com"
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

At this point. We can add a CNAME record on DNS Management.

```SHELL
RECORD  ADDRESS           DEST                                                TTL
CNAME   mydomain.com      IP-of-Nginx-Reverse-proxy                           300
```
