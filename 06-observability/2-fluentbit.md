## Configure fluentbit to send logs to cloudwatch

1. Create a namespace named `amazon-cloudwatch`
2. Create a configmap that defines the fluentbit
```
ClusterName=<cluster-name>
RegionName=<cluster-region>
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
kubectl create configmap fluent-bit-cluster-info \
--from-literal=cluster.name=${ClusterName} \
--from-literal=http.server=${FluentBitHttpServer} \
--from-literal=http.port=${FluentBitHttpPort} \
--from-literal=read.head=${FluentBitReadFromHead} \
--from-literal=read.tail=${FluentBitReadFromTail} \
--from-literal=logs.region=${RegionName} -n amazon-cloudwatch

```


3. Create an IAM role with the following trust-policy

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:amazon-cloudwatch:fluent-bit"
        }
      }
    }
  ]
}
```
3. Add the managed permission policy `CloudWatchAgentServerPolicy`


4. Annotate the service account

```
kubectl annotate serviceaccount \
  fluent-bit \
  -n amazon-cloudwatch \
  eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/<YourFluentBitRole> \
  --overwrite

```

3.  Install fluentbit

```
kubectl apply -f https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/fluent-bit/fluent-bit.yaml
```


4. create a deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-logger
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-logger
  template:
    metadata:
      labels:
        app: nginx-logger
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

      - name: logger
        image: bash:latest
        command: ["bash", "-c"]
        args:
          - |
            while true; do
              echo "the time is $(date)";
              sleep 5;
            done
```

