## Create a VPC for the EKS cluster

1. Create a vpc with /16 CIDR range : 192.168.0.0/16
2. Create an internet gateway and attach to the vpc
3. Create 2 public subnets with the following cidrs: `192.168.0.0/18` and `192.168.64.0/18` and add the following tags:
   ```
   kubernetes.io/role/elb = 1
   ```
4. Create 2 private subnets with the following cidrs: `192.168.128.0/18` and `192.168.192.0/18` and add the following tags:
```
kubernetes.io/role/internal-elb = 1
```
5. Make sure the internet gateway is attached to the vpc we created earlier
6. Create a NAT gateway in one of the public subnets
7. Create a public route table and a private route table
8. Add the following route to the public route table
   
```
to : 0.0.0.0/0
via: internet gateway
```
9. Add the following route to the private route table

```
to: 0.0.0.0/0
via: NAT gateway
```

10. Associate the public route table with the public subnets and private route table with private subnets.

## Create an iam role 

1. Create an iam policy - eks-cluster-role-trust-policy.json
   
```

 {
  "Version": "2012-10-17",		 	 	 
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}

```

2. Create an IAM role (either through aws management console or using ) with the trust relationship created above.
3. Assign AWS managed permission policy - AmazonEKSClusterPolicy
4. Create cluster using the documentation - [create cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html#step2-console)
