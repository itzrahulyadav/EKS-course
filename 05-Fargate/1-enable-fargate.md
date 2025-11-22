# Enable aws fargate


## Create a role

1. Create a role called `AmazonEKSFargatePodExecutionRole`
2. for the trust relationship use  AWS service -> EKS -> EKS - Fargate Pod.
3. Attach the managed policy to the role - AmazonEKSFargatePodExecutionRolePolicy



## Create a fargate profile
1. Create a fargate profile with namespace and labels

```
namespace: prod
labels:
  env: prod
```

3. Now all the pods that are created in the mentioned namespace and with the selected labels will run on fargate
4. Create a deployment that matches the fargate profile

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: prod
  labels:
    env: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      env: prod
  template:
    metadata:
      labels:
        env: prod
    spec:
      containers:
      - name: app
        image: nginx:alpine  
        ports:
        - containerPort: 80

```


## Enable logging

1. Create a namespace called aws-observability (mandatory)

```
kind: Namespace
apiVersion: v1
metadata:
  name: aws-observability
  labels:
    aws-observability: enabled

```

- Create log-group

2. Create a configmap that specifies which logs to send

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: aws-logging
  namespace: aws-observability
data:
  flb_log_cw: "false"  # Set to true to ship Fluent Bit process logs to CloudWatch.
  filters.conf: |
    [FILTER]
        Name parser
        Match *
        Key_name log
        Parser crio
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
        Buffer_Size 0
        Kube_Meta_Cache_TTL 300s
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match   kube.*
        region <region>
        log_group_name <log group>
        log_stream_prefix from-fluent-bit-
        log_retention_days 60
        auto_create_group true
  parsers.conf: |
    [PARSER]
        Name crio
        Format Regex
        Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>P|F) (?<log>.*)$
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L%z

```

3. Add permissions to the role

```
curl -O https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json

```

```
aws iam create-policy --policy-name eks-fargate-logging-policy --policy-document file://permissions.json

```

```
aws iam attach-role-policy \
  --policy-arn <policy> \
  --role-name AmazonEKSFargatePodExecutionRole

```




