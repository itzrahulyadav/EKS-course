Kyverno is a Kubernetes-native policy engine that allows you to define security, governance, and configuration policies for Kubernetes resources using YAML instead of code

- [Kyverno Docs](http://kyverno.io/)

Some of the rules that can be enforced using Kyverno.
 
 - preventing containers from running as root
 - Pods must have resource limits
 - Allow only trusted container registries
 - Add labels or annotations

Kyverno is basically a admission controller that intercepts kubernetes API requests and validate or mutate them.

```

Developer -> kubectl apply -> Kubernetes API Server
                                |
                                v
                          Kyverno Admission Controller
                                |
               -------------------------------------
               |                                   |
        Policy Pass                         Policy Fail
               |                                   |
         Resource Created                  Request Rejected

```


## Using kyverno in EKS Cluster

1. Install kyverno.

```
kubectl create namespace kyverno

helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno -n kyverno --create-namespace

```

2. Check kyverno pods

```
kubectl get pods -n kyverno

```

## Demo 1

1. Block privileged containers

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: block-privileged-containers
spec:
  validationFailureAction: Enforce
  rules:
  - name: privileged-containers
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Privileged containers are not allowed"
      pattern:
        spec:
          containers:
          - =(securityContext):
              =(privileged): "false"

```

2. Try creating a pod

```
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
```



# Demo 2

Force Resource limits

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-resources
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "CPU and memory limits are required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
```

2. Try creating a pod


# Demo 3

Mutate the pods

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-env-label
spec:
  rules:
  - name: add-label
    match:
      resources:
        kinds:
        - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            environment: dev
```

2. Try creating a pod, labels will be added automatically


# Demo 4

Allow only trusted container registries

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: trusted-registry
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-registry
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Images must come from ECR"
      pattern:
        spec:
          containers:
          - image: "*.amazonaws.com/*"

```

