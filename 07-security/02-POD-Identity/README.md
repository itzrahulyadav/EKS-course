# Enable Pod identity in EKS Cluster

1. Install the addon

```
eksctl create addon \
  --name eks-pod-identity-agent \
  --cluster clusterName \
  --region region
```

2.  Check the pods

```
kubectl get pods -n kube-system | grep pod-identity

```

3. Create a role with the following trust policy

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```
4. Create a role using the following command

```
aws iam create-role \
  --role-name eks-pod-identity-s3-role \
  --assume-role-policy-document file://trust.json

```

5. Attach a policy to the role - ` AmazonS3FullAccess `


6. Create a service account by creating a file called sa.yaml and then applying the file `kubectl apply -f sa.yaml `

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: default
```

7. Create a pod identity association

```
aws eks create-pod-identity-association \
  --cluster-name my-cluster \
  --namespace default \
  --service-account <service account> \
  --role-arn <role_name>
```

8. Verify the assocation

```
aws eks list-pod-identity-associations \
  --cluster-name <cluster name>
```

9. Create a pod.yaml file and apply the file using ` kubectl apply -f pod.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: s3-test
spec:
  serviceAccountName: s3-reader
  containers:
  - name: awscli
    image: amazon/aws-cli
    command: ["sleep", "3600"]
```

10. Exec into the pod and check the s3 accessibility

```
kubectl exec -it <pod_name> -- sh

```
