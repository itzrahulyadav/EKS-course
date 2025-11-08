## Create a deployment called blue deployment 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: blue
  template:
    metadata:
      labels:
        app: blue
    spec:
      containers:
        - name: blue
          image: nginx
          ports:
            - containerPort: 80

```

## Create a red deployment with affinity for blue deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: red-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: red
  template:
    metadata:
      labels:
        app: red
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - blue
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: red
          image: nginx
          ports:
            - containerPort: 80
```

## Create a deployment with node anti-affinity for blue deployment

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: red-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: red
  template:
    metadata:
      labels:
        app: red
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - blue
              topologyKey: "kubernetes.io/hostname"
      containers:
        - name: red
          image: nginx
          ports:
            - containerPort: 80

```
