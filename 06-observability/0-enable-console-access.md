## run the following commands to allow cluster access to the console users


1. Create acess entry

```
aws eks create-access-entry \
  --cluster-name <clustername> \
  --principal-arn arn:aws:iam::<account_id>:root \
  --type STANDARD

```

2. Add the access-policies

```
aws eks associate-access-policy \
  --cluster-name <cluster_name> \
  --principal-arn <principal arn> \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

```
