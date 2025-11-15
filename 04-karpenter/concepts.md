## Karpenter


[Official docs](https://karpenter.sh/docs/getting-started/getting-started-with-karpenter/)

Before setting up Karpenter, let’s cover key terms you’ll encounter:


###  NodePool:

A Kubernetes Custom Resource Definition (CRD) that defines the constraints for node provisioning, such as instance types, architectures, operating systems, and capacity types (e.g., On-Demand or Spot). It specifies how Karpenter selects nodes for pods.
Example: A NodePool might allow c5.large or m5.xlarge instances in us-west-2a with Spot priority.

### EC2NodeClass:
Another CRD that defines AWS-specific configurations for nodes, such as the IAM role, AMI (Amazon Machine Image), subnets, and security groups.
Example: Links to an IAM role (KarpenterNodeRole) and specifies an Amazon Linux 2023 AMI.
Provisioner (Deprecated in v1+):
In older Karpenter versions (v0.x), a Provisioner was used instead of NodePool and EC2NodeClass. It combined node selection and cloud provider settings. In Karpenter v1+, NodePool and EC2NodeClass replace it for better modularity.
### NodeClaim:
A CRD representing a request for a new node. When Karpenter detects unschedulable pods, it creates a NodeClaim to provision a matching EC2 instance, which becomes a Kubernetes node.
Consolidation:
Karpenter’s process of moving pods to fewer nodes to optimize resource usage, terminating underutilized nodes. It uses a bin-packing algorithm to pack pods efficiently.

### Disruption:
The process of terminating nodes when they’re no longer needed or to consolidate workloads. Controlled by policies like consolidationPolicy: WhenEmptyOrUnderutilized and consolidateAfter.

###  Capacity Type:
Specifies whether nodes use On-Demand or Spot instances. Spot Instances are cheaper but can be interrupted, while On-Demand ensures stability. Karpenter prioritizes cost-effective options.

### Node Affinity:
Kubernetes scheduling constraints (e.g., nodeSelector, taints, tolerations) that Karpenter respects when provisioning nodes to match pod requirements.

### Karpenter Controller:
The Kubernetes deployment that runs Karpenter, monitoring unschedulable pods and managing node lifecycle via AWS EC2 APIs.

### Spot Instances:
AWS EC2 instances offered at a discount but subject to interruption. Karpenter’s support for Spot Instances enhances cost savings.
TTL (Time to Live):
Settings like expireAfter (e.g., 720h for 30 days) or ttlSecondsAfterEmpty (e.g., 30s) control when nodes are terminated due to age or underutilization.

###  Bin-Packing:
Karpenter’s algorithm to efficiently pack pods onto nodes, minimizing resource waste and reducing the number of nodes needed.
