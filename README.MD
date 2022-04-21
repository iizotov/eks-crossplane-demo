# Crossplane.io on EKS - quick demo

## Install crossplane CLI
```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
```

## Install crossplane using Helm
```
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

## Validate installation - all must be healthy
```
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

## Install config package
```
kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-aws:v1.7.0
```

## Wait until it goes healthy - give it a minute or so
```
kubectl get pkg
```

## Demo only - create user for crossplane - normally you will be using IAM
```
CPUSER="crossplane"
aws iam create-user --user-name $CPUSER
aws iam attach-user-policy --user-name $CPUSER --policy-arn "arn:aws:iam::aws:policy/AdministratorAccess"
TMP=($(aws iam create-access-key --user-name $CPUSER --output text))
echo -e "[default]\naws_access_key_id = ${TMP[1]}\naws_secret_access_key = ${TMP[3]}" > creds.conf
```

## Add these credentials as a secret to kubernetes
```
kubectl create secret generic aws-creds -n crossplane-system --from-file=creds=./creds.conf
```

## Configure the AWS Provider to use them
```
cat <<EOF | kubectl create -f -
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
---
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: aws-provider-config
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
EOF
```

## Let's test! 
We'll use https://aws.amazon.com/blogs/opensource/introducing-aws-blueprints-for-crossplane/
```
cd ~/environment/
git clone https://github.com/aws-samples/crossplane-aws-blueprints
cd crossplane-aws-blueprints

```

## Deploy compositions
```
kubectl apply -f compositions/aws-provider/dynamodb 
```

## Check if ready to be used
```
kubectl get xrd xdynamodbtables.awsblueprints.io 
```

## Deploy DynamoDB in us-west-2
```
kubectl apply -f examples/aws-provider/composite-resources/dynamodb/dynamodb-on-demand-partition.yaml
```

## Check status - it will take a couple of minutes
Also check out the console https://us-west-2.console.aws.amazon.com/dynamodbv2/home?region=us-west-2#tables
```
kubectl get dynamodbtable.awsblueprints.io/test-table-on-demand-partition-key
```

## Let's provision an RDS instance in default VPC in us-east-1
```
cat <<EOF | kubectl create -f -
apiVersion: database.example.org/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-db
  namespace: default
spec:
  parameters:
    storageGB: 20
  compositionSelector:
    matchLabels:
      provider: aws
      vpc: default
  writeConnectionSecretToRef:
    name: db-conn
EOF
```

## Check status of provisioning - it will take around 5 minutes to provision
look into the console too - https://us-east-1.console.aws.amazon.com/rds/home?region=us-east-1#databases:
```
kubectl get postgresqlinstance my-db
```

## Clean up
```
kubectl delete postgresqlinstance my-db
kubectl delete dynamodbtable.awsblueprints.io/test-table-on-demand-partition-key
```
