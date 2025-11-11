## Installing karpenter in EKS Cluster

1. Create a trust-policy.json file

```
cat <<EOF > controller-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$AWS_ACCOUNT_ID:oidc-provider/${OIDC_ENDPOINT#*//}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
          "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:kube-system:karpenter"
        }
      }
    }
  ]
}

```

2.  Run the following command

```
aws iam create-role --role-name KarpenterControllerRole --assume-role-policy-document file://controller-trust-policy.json
```
3. Create a permission policy

```
cat <<EOF > controller-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ec2:DescribeImages",
        "ec2:RunInstances",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeLaunchTemplates",
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeInstanceTypeOfferings",
        "ec2:DeleteLaunchTemplate",
        "ec2:CreateTags",
        "ec2:CreateLaunchTemplate",
        "ec2:CreateFleet",
        "ec2:*"
        "ec2:DescribeSpotPriceHistory",
        "pricing:GetProducts"
        "iam:*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "ec2:TerminateInstances",
      "Condition": {
        "StringLike": {
          "ec2:ResourceTag/karpenter.sh/nodepool": "*"
        }
      },
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "<arn of the role used by nodegroup>"
    },
    {
      "Effect": "Allow",
      "Action": "eks:DescribeCluster",
      "Resource": "<eks cluster arn>"
    }
  ]
}

```

4. Add the policy to the role created in step 2

```
aws iam put-role-policy --role-name KarpenterControllerRole --policy-name KarpenterControllerPolicy --policy-document file://controller-policy.json
```

5.  Install karpenter using helm
Replace the fields according to your cluster

```
export KARPENTER_VERSION="1.0.8"

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace kube-system \
  --version $KARPENTER_VERSION \
  --set settings.clusterName=<CLUSTER_NAME> \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=<karpenter controller role> \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=1Gi \
  --set controller.resources.limits.cpu=1 \
  --set controller.resources.limits.memory=1Gi \
  --create-namespace
```

6.  Tag security groups and subnets so karpenter can use them

```
aws ec2 create-tags --resources <subnet-id-1> <subnet-id-2> --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME
aws ec2 create-tags --resources <security-group-id> --tags Key=karpenter.sh/discovery,Value=$CLUSTER_NAME

```

7.  Find the optimized AMIs for your region

```
aws ssm get-parameter --name /aws/service/eks/optimized-ami/1.31/amazon-linux-2023/x86_64/standard/recommended/image_id --region ap-south-1 --query Parameter.Value --output text
```

8. Configure nodepool and EC2NodeClass

```
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: <role created for ec2 nodes of karpenter or eks cluster nodegroups>
  amiSelectorTerms:
    - id: ami-09f1941e8730d7702 
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: <eks cluster name>
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: <eks cluster name>

```

9. Troubleshooting in case the Karpenter pods don't spin up or karpenter does not launch new nodes for pods

```
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter | grep -i error
kubectl get nodepools -w
kubectl get pods -n default -w
kubectl get nodes -w
kubectl get nodeclaims -n kube-system -w
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter -f

kubectl get clusterrole karpenter -o yaml

kubectl get nodepools
kubectl describe nodepool default
kubectl describe ec2nodeclass default

kubectl describe pod -n kube-system -l app.kubernetes.io/name=karpenter

```


10. In case you created a separate role for ec2 nodes that will be launched by karpenter , add this access entry in the cluster

```

aws eks create-access-entry \
  --cluster-name eks-cluster \
  --principal-arn arn:aws:iam::<account-id>:role/KarpenterNodeRole-eks-cluster \
  --type STANDARD \
  --region ap-south-1
aws eks associate-access-policy \
  --cluster-name eks-cluster \
  --principal-arn arn:aws:iam::<account-id>:role/KarpenterNodeRole-eks-cluster \
  --access-scope type=cluster \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSNodeAccess \
  --region ap-south-1

```

11. Best of Luck

