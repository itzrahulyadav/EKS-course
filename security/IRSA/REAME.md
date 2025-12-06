# Use IRSA to access AWS resources

1. Create a pod `pod.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: default
spec:
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    command: [ "sleep", "3600" ]
```

2. Run the following command to check the credentials

```
kubectl exec -it aws-cli -- bash

aws sts get-caller-identity

```

3.  Create a service account `sa.yaml`

```

apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-sa
  namespace: default

```

4. Create an IAM role with the following trust relationship `trust-policy.json`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_PROVIDER"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$OIDC_PROVIDER:sub": "system:serviceaccount:default:s3-sa"
        }
      }
    }
  ]
}

```

5.  Create IAM role

```
aws iam create-role \
  --role-name demo-irsa-role \
  --assume-role-policy-document file://trust-policy.json

```

6.  Create a permission policy

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [ "s3:ListAllMyBuckets" ],
      "Resource": "*"
    }
  ]
}

```

5. Attach the permissions to the IAM role

```
aws iam attach-role-policy \
  --role-name demo-irsa-role \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/demo-irsa-s3-list-policy

```

6. Annotate the service account

```
kubectl annotate sa demo-sa \
  eks.amazonaws.com/role-arn=arn:aws:iam::$ACCOUNT_ID:role/demo-irsa-role \
  --overwrite

```



Your service account would look something like this after annotating it

```
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2025-01-01T12:00:00Z"
  name: s3-sa
  namespace: default
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/demo-irsa-role
  uid: 12345678-1234-1234-1234-1234567890ab
secrets:
- name: demo-sa-token-abcde

```



7.  Delete and recreate the pods

```
kubectl exec -it aws-cli -- bash
aws sts get-caller-identity
aws s3 ls

```

8. Delete the cloudformation stack
