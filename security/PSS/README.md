# Privileged PSS

## Create a deployment 

1. Create a file called privileged-demo.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: privileged-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: privileged-demo
  template:
    metadata:
      labels:
        app: privileged-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        securityContext:
          privileged: true

```

2. Apply the file

```
kubectl apply -f privileged-demo.yaml
```

3. Exec into pods

```
kubectl get pods
kubectl exec -it <pod-name> -- /bin/bash

```

4. Run this command

```
id

```

## Baseline PSS

1. Label the namespace

```
kubectl label namespace default \
pod-security.kubernetes.io/enforce=baseline

kubectl get ns default --show-labels

```

2. Apply the file again - Recreate the deployment

```
kubectl apply -f privileged-demo.yaml

```


## Restricted PSS

1. Label the namespace

```
kubectl label namespace default \
pod-security.kubernetes.io/enforce=baseline

kubectl get ns default --show-labels

```

2. Trying create a simple pod



