1. Create a service of type clusterIP
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-demo
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP

```

2. Create ingress to expose our application to the internet using alb
   
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    alb.ingress.kubernetes.io/group.name: demo-group
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80


```


## Expose multiple applications through the same ingress

1. Create a deployment
```
# app2-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: app2
        image: itzrahulyadav/eks-demo
        ports:
        - containerPort: 8080
```

2. Create a service

```
# app2-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  type: ClusterIP
  selector:
    app: app2
  ports:
    - port: 80
      targetPort: 80

```

3. Create an ingress and use the same group name

```
# app2-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app2-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    alb.ingress.kubernetes.io/group.name: demo-group
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: app2-service
                port:
                  number: 80

```
