

Pod Security Standards (PSS) are a set of predefined security profiles in Kubernetes that define how “restricted” a Pod can be in terms of security settings.

They were introduced to replace PodSecurityPolicy

# Privileged PSS

Privileged (Least Restrictive)

- Allows almost everything
- For system-level workloads
- Can run as root
- Can use host networking, host PID, hostPath, etc.

Use case:
- CNI plugins
- CSI drivers
- Node-level agents

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


Prevents common privilege escalation:

- ❌ No hostPID / hostIPC
- ❌ Restricted hostPath usage
- ❌ No dangerous Linux capabilities
- ❌ No privileged containers


But: May still run as root

Use case: Most regular application workloads


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

You will get a warning stating:

```
Error from server (Forbidden): 
pods "privileged-demo-xxxxx" is forbidden: 
violates PodSecurity "baseline:latest":
privileged (container "nginx" must not set securityContext.privileged=true)

```

## Restricted PSS

Very strict:
- ❌ Must use seccomp
- ❌ Limited volume types
- ❌ No privileged containers
- ❌ No privilege escalation
- ❌ Must NOT run as root

Use case:
- Production workloads
- Multi-tenant clusters

1. Label the namespace

```
kubectl label namespace default \
pod-security.kubernetes.io/enforce=restricted --overwrite

kubectl get ns default --show-labels

```

2. Try to create a simple pod

Warnings :

```
Warning: would violate PodSecurity "restricted:latest": privileged (container "nginx" must not set securityContext.privileged=true), allowPrivilegeEscalation!= false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")deployment.apps/privileged-demo created

```


Try modifying the file: (run nginx as non root user and modified capabilities)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: safe-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: safe-nginx
  template:
    metadata:
      labels:
        app: safe-nginx
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: nginx
        image: bitnami/nginx:latest
        ports:
        - containerPort: 8080
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          capabilities:
            drop:
              - ALL

```


