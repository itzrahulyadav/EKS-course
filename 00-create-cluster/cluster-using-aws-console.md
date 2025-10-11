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
5. Create an internet gateway and attach it to the vpc we created earlier
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
5. Install kubectl,helm and eksctl using the script

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
