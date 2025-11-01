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

- Install the EFS add-on on the cluster using the role created

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

11. Create storage class for efs

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
allowVolumeExpansion: true
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-92107410
  directoryPerms: "700"

```

12.  Create persistent volume claim

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
```

13. Create a pod

```
apiVersion: v1
kind: Pod
metadata:
  name: efs-app
spec:
  containers:
    - name: app
      image: nginx
      command: ["/bin/sh"]
      args: ["-c", "while true; do echo $(date -u) >> /data/out; sleep 10; done"]
      volumeMounts:
        - name: persistent-storage
          mountPath: /data
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: efs-claim

```

14. Create a deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: persistent-storage-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-persistent-app
  template:
    metadata:
      labels:
        app: my-persistent-app
    spec:
      containers:
      - name: app-container
        image: nginx
        volumeMounts:
        - name: persistent-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: persistent-storage
        persistentVolumeClaim:
          claimName: efs-claim

```

