# Create an IAM role for managed nodes

1. Follow the documentation to create role using aws managed console - [Create node role](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role)
2. Add the following permission policies
   - AmazonEKSWorkerNodePolicy
   - AmazonEC2ContainerRegistryReadOnly
   - AmazonEKS_CNI_Policy
   - AmazonSSMManagedInstanceCore

3. Create the nodegroups in private subnet 
