## Install load balancer controller in our cluster

1. Install the IAM policy file needed by the controller iam role
   
```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.14.0/docs/install/iam_policy.json

```

2. Create an IAM policy using the file downloaded in the previous step

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

```

3. Crete a trust policy file with the following content - trust-policy.json

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "<OIDC_ARN>"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "<Provider_name>:sub": "system:serviceaccount:kube-system:<service_account_name>"
                }
            }
        }
    ]
}

```

4. Create role using cli command
```
aws iam create-role --role-name AWSLoadBalancerControllerRole --assume-role-policy-document file://trust-policy.json --region ap-south-1

```
5. Attach the IAM policy that was created

```
aws iam attach-role-policy --role-name AWSLoadBalancerControllerRole --policy-arn <policy_arn> --region ap-south-1

```


6. create Service account using the following manifest
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: <load balancer controller role arn>

```

7. Install aws load balancer controller using helm

```
helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster_name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=<vpc_id> \
  --version 1.13.0

```

8. Check load balancer controller logs

```
k -n kube-system logs -f --selector app.kubernetes.io/instance=aws-load-balancer-controller

```
