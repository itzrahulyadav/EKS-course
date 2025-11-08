## Create a deployment

```
kubectl create deployment stress-test --image=busybox -- sleep 100000
```

# Create a role for cluster autoscaler

1. Create a `trust-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:kube-system:cluster-autoscaler-sa",
          "<OIDC_PROVIDER>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}

```
2.  Create a role using the trust-policy created above
   
```
aws iam create-role \
  --role-name AutoscalerRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "IAM role for Cluster Autoscaler"

```

3. Use the following permission policy and create a file called `cluster-autoscaler-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": ["*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": ["*"]
    }
  ]
}

```

4. Create a cluster-policy

```
aws iam create-policy \
  --policy-name AmazonEKSClusterAutoscalerPolicy \
  --policy-document file://cluster-autoscaler-policy.json
  
```

5. Attach the policy to the role

```
aws iam attach-role-policy \
  --role-name AutoscalerRole \
  --policy-arn arn:aws:iam::<account-id>:policy/AmazonEKSClusterAutoscalerPolicy

```

5.1 Create service-account 

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-autoscaler-sa
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: <Auto scaler role arn>

```

6.  Add the helm repo autoscaler

```
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

```

7. Install the autoscaler helm chart

```
helm upgrade --install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=<your-cluster-name> \
  --set awsRegion=<your-region> \
  --set cloudProvider=aws \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler-sa \
  --set extraArgs.balance-similar-node-groups=true \
  --set extraArgs.skip-nodes-with-system-pods=false

```

9. kubectl -n kube-system get pods | grep cluster-autoscaler

10. Let's scale our deployment

```
kubectl scale deployment stress-test --replicas=20

```

