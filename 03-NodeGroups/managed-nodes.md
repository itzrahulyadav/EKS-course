# Check the nodegroup of the Cluster

```
eksctl get nodegroup --cluster <cluster_name> --name <node_group_name>

```

# See the nodes placement

```
kubectl get nodes -o wide --label-columns topology.kubernetes.io/zone
```

## Change the nodegroups size

```
aws eks update-nodegroup-config --cluster-name <cluster_name> \
  --nodegroup-name <nodegroup_name> --scaling-config minSize=2,maxSize=3,desiredSize=3

```

## NodeGroups can also be updated through the management console.
