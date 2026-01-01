# Create a secret in your kubernetes cluster

```
kubectl create secret generic app-secrets \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=supersecret

```

2. Check the secret

```
kubectl get secrets

```

3. Create a deployment that uses the secrets

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secret-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secret-demo
  template:
    metadata:
      labels:
        app: secret-demo
    spec:
      containers:
      - name: nginx
        image: nginx:alpine

        # Inject secrets as env vars
        env:
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: DB_PASSWORD

        # Override default nginx command
        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "🔐 Checking secrets..."

            if [ -z "$DB_USER" ] || [ -z "$DB_PASSWORD" ]; then
              echo "❌ Secrets not found! Exiting..."
              exit 1
            fi

            echo "✅ Secrets loaded successfully"
            echo "DB_USER=$DB_USER"
            echo "DB_PASSWORD=$DB_PASSWORD"

            echo "🚀 Starting nginx..."
            nginx -g 'daemon off;'

```

4. Delete the secret

```
kubectl delete secret app-secrets

```

5.  Restart the deployment and see the pod fails to start

```
kubectl rollout restart deploy/secret-demo

```
## Using secrets manager instead of local secrets.


1.  Create secret

```
aws secretsmanager create-secret \
  --name demo/nginx/credentials \
  --secret-string '{
    "username": "admin",
    "password": "supersecret"
  }'

```



2.  Create IAM Role for ASCP (IRSA or Pod Identity)

- use the following trust policy.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/<OIDC_PROVIDER>"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "<OIDC_PROVIDER>:sub": "system:serviceaccount:default:get-secret-sa",
          "<OIDC_PROVIDER>:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}

```

- Add the following permissions

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:demo/nginx/credentials*"
    }
  ]
}

```

3. Install the add-on

```
aws eks update-addon \
  --cluster-name <cluster_name> \
  --addon-name secrets-store-csi-driver


```

4.  Install ascp provider

```
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

```

5. Check the pods

```
kubectl get pods -n kube-system | grep secrets-store

```

6. Create a secretprovider class

```
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets-demo
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "demo/nginx/credentials"
        objectType: "secretsmanager"
        jmesPath:
          - path: username
            objectAlias: DB_USER
          - path: password
            objectAlias: DB_PASSWORD


```

 Create a service-account
 
```
kubectl create sa get-secret-sa

```

Annotate the service account

```
kubectl annotate serviceaccount get-secret-sa eks.amazonaws.com/role-arn=<role_arn>

```




7.  Create a deployment with the new configs


```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ascp-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ascp-demo
  template:
    metadata:
      labels:
        app: nginx-ascp-demo
    spec:
      serviceAccountName: get-secret-sa
      containers:
      - name: nginx
        image: nginx:alpine

        volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true

        command: ["/bin/sh", "-c"]
        args:
          - |
            echo "🔐 Reading secrets from AWS Secrets Manager..."

            DB_USER=$(cat /mnt/secrets/DB_USER 2>/dev/null)
            DB_PASSWORD=$(cat /mnt/secrets/DB_PASSWORD 2>/dev/null)

            if [ -z "$DB_USER" ] || [ -z "$DB_PASSWORD" ]; then
              echo "❌ Failed to fetch secrets from AWS"
              exit 1
            fi

            echo "✅ Secrets fetched successfully"
            echo "DB_USER=$DB_USER"
            echo "DB_PASSWORD=$DB_PASSWORD"

            nginx -g 'daemon off;'

      volumes:
      - name: secrets-store
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: aws-secrets-demo

```

8. Check the logs

```
kubectl logs deploy/nginx-ascp-demo

```


