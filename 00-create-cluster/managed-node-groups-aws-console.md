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

6. Create an ec2 instance in the same vpc as your EKS cluster.
7. Install kubectl,helm and eksctl using the script

```
#!/bin/bash
# Log all output to /var/log/user-data.log
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
          
# Update packages
dnf update -y

# Download kubectl
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.0/2025-05-01/bin/linux/amd64/kubectl
curl -o kubectl.sha256 https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.0/2025-05-01/bin/linux/amd64/kubectl.sha256
                                
# Verify checksum
sha256sum -c kubectl.sha256 || exit 1
          
# Install kubectl
chmod +x ./kubectl
mkdir -p /root/bin
cp ./kubectl /root/bin/kubectl
echo 'export PATH=/root/bin:$PATH' >> /root/.bashrc
echo 'export PATH=/root/bin:$PATH' >> /root/.bash_profile
kubectl version --client

# Install eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check || exit 1
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
mv /tmp/eksctl /usr/local/bin/eksctl
eksctl version
          
# Install helm
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh
helm version

```

7. Update the kubeconfig using the following command

```
aws eks update-kubeconfig --region ap-south-1 --name <my_cluster>

```

