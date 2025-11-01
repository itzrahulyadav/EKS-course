# Create an IAM role with the necessary permissions

1.  Create trust-policy.json

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
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa",
          "<OIDC_PROVIDER>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

2. Create the IAM role with the above trust-policy as trust policy

```
aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file://trust-policy.json \
  --description "IAM role for EBS CSI driver in EKS"
```

3. Attach the managed policy for permission

```
aws iam attach-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

```
- Install the EBS add-on on the Cluster using the role created

4. Create a storage class
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3  # Or io2 for high IOPS, etc.
  encrypted: "true"  # Optional: Enables default EBS encryption
volumeBindingMode: WaitForFirstConsumer  # Recommended: Binds volume after pod is scheduled (handles AZ affinity)
reclaimPolicy: Delete  # Deletes volume when PVC is deleted; use "Retain" for persistence
allowedTopologies:  # Optional: Restrict to specific AZs
  - matchLabelExpressions:
      - key: topology.ebs.csi.aws.com/zone
        values:
          - us-west-2a  # Example AZ

```

5. Make it the default storage class

```

kubectl patch storageclass ebs-gp3-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

```
6. Create pvc

```

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-pvc
spec:
  accessModes:
    - ReadWriteOnce  # EBS supports single-node access
  storageClassName: ebs-gp3-sc
  resources:
    requests:
      storage: 5Gi  

```

7. Create deployment.yaml

```

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ebs-app
spec:
  replicas: 1  # Ensure only one pod
  selector:
    matchLabels:
      app: ebs-app
  template:
    metadata:
      labels:
        app: ebs-app
    spec:
      nodeSelector:
        topology.kubernetes.io/zone: ap-south-1a  # Target ap-south-1a
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo 'Data on EBS' >> /data/file.txt; sleep 5; done"]
        volumeMounts:
        - mountPath: /data
          name: ebs-volume
      volumes:
      - name: ebs-volume
        persistentVolumeClaim:
          claimName: ebs-pvc


```
