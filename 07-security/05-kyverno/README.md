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

kubectl api-resources | grep kyverno

```

2. Check kyverno pods

```
kubectl get pods -n kyverno

```

## Demo 1

1. Block privileged containers

```
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: disallow-privileged
spec:
  validationActions:
    - Deny
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["pods"]
  validations:
  - expression: "object.spec.containers.all(c, !has(c.securityContext) || c.securityContext.privileged == false)"
    message: "Privileged containers are not allowed"

```
Verify using this command: `kubectl get validatingpolicies`

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

Auto create network policies

```
apiVersion: policies.kyverno.io/v1
kind: GeneratingPolicy
metadata:
  name: default-network-policy
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["namespaces"]

  variables:
    - name: nsName
      expression: "object.metadata.name"

    - name: downstream
      expression: >-
        [
          {
            "kind": dyn("NetworkPolicy"),
            "apiVersion": dyn("networking.k8s.io/v1"),
            "metadata": dyn({
              "name": "default-deny",
              "namespace": string(variables.nsName)
            }),
            "spec": dyn({
              "podSelector": dyn({}),
              "policyTypes": dyn(["Ingress", "Egress"])
            })
          }
        ]

  generate:
    - expression: generator.Apply(variables.nsName, variables.downstream)

```

2. Try creating a pod


# Demo 3

Mutate the pods

```
apiVersion: policies.kyverno.io/v1
kind: MutatingPolicy
metadata:
  name: add-label
spec:
  matchConstraints:
    resourceRules:
      - apiGroups: ['']
        apiVersions: ['v1']
        operations: ['CREATE']
        resources: ['pods']
  mutations:
    - patchType: ApplyConfiguration
      applyConfiguration:
        expression: >
          Object{
            metadata: Object.metadata{
              labels: Object.metadata.labels{
                foo: "bar"
              }
            }
          }

```

2. Try creating a pod, labels will be added automatically


# Demo 4

Allow only trusted container registries

```
apiVersion: policies.kyverno.io/v1
kind: ValidatingPolicy
metadata:
  name: allow-only-ecr
spec:
  validationActions:
    - Deny

  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]

  validations:
    - expression: >
        object.spec.containers.all(c,
          c.image.matches("^[0-9]+\\.dkr\\.ecr\\.[a-z0-9-]+\\.amazonaws\\.com/.*")
        )
      message: "Only AWS ECR images are allowed"

```

