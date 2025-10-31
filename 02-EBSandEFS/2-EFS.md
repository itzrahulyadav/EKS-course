1. Create a role with the following trust policy : efs-trust-policy.json

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
        "StringLike": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:kube-system:efs-csi-*",
          "<OIDC_PROVIDER>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}

```

2. Create an IAM role using the following command

```
aws iam create-role \
  --role-name AmazonEKS_EFS_CSI_DriverRole \
  --assume-role-policy-document file://"efs-trust-policy.json"
```

3. Attach the managed policy to the role

```
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy \
  --role-name AmazonEKS_EFS_CSI_DriverRole

```

4. Note down the following details to create a efs volume which will be used for dynamic volume provisioning
 - VPC id of the cluster
 - CIDR range of the VPC

5. Create a security group for you EFS file system
6.  Allow inbound traffic on port 2049 from the EKS CIDR range that we noted earlier
7.  Create an EFS file system in the same region as your EKS cluster

```
file_system_id=$(aws efs create-file-system \
    --region region-code \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)

```

9.  Note down the subnets and AZs of your eks nodes
10.  Create mount target in each subnets

```
aws efs create-mount-target \
    --file-system-id <file_system_id> \
    --subnet-id <subnet_id> \
    --security-groups <security_group_id>

```
