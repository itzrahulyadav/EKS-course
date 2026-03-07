Kyverno is a Kubernetes-native policy engine that allows you to define security, governance, and configuration policies for Kubernetes resources using YAML instead of code

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




