# Create an IAM role for managed nodes

1. Follow the documentation to create role using aws managed console - [Create node role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role)
2. Add the following trust relationship

```
{
    "Version": "2012-10-17",		 	 	 
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Principal": {
                "Service": [
                    "ec2.amazonaws.com"
                ]
            }
        }
    ]
}
```

3. Add the following permission policies
   - AmazonEKSWorkerNodePolicy
   - AmazonEC2ContainerRegistryReadOnly
   - AmazonEKS_CNI_Policy
   - AmazonSSMManagedInstanceCore

5. Create the nodegroups in private subnet 
