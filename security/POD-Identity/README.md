# Enable Pod identity in EKS Cluster

1. Install the addon

```
eksctl create addon \
  --name eks-pod-identity-agent \
  --cluster clusterName \
  --region region
```

2.  
