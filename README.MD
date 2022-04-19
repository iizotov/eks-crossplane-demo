# Crossplane.io on EKS - quick demo

### 1. Install crosplane.io
```
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --create-namespace --namespace crossplane-system crossplane-stable/crossplane
```

### 2. Check status
```
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

### 3. Install Crossplane CLI
```
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /usr/local/bin
kubectl crossplane --version
kubectl crossplane --help
```

### 3. Let's enable identity for crossplane
```
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
IAM_ROLE_NAME=provider-aws # name for IAM role, can be anything you want

cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1alpha1
kind: ControllerConfig
metadata:
  name: aws-config
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::$AWS_ACCOUNT_ID:role/$IAM_ROLE_NAME
spec:
  podSecurityContext:
    fsGroup: 2000
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: crossplane/provider-aws:v${VERSION}
  controllerConfigRef:
    name: aws-config
EOF
```

### 4. Make sure that the appropriate `ServiceAccount` exists
```
kubectl get serviceaccounts -n crossplane-system
```
You should see a ServiceAccount in the output named provider-aws-*. You should also see the provider-aws-* controller Pod running if you execute kubectl get pods -n crossplane-system. Set variables to match the name and namespace of this ServiceAccount:

```
SERVICE_ACCOUNT_NAMESPACE=crossplane-system
SERVICE_ACCOUNT_NAME=$(kubectl get providers.pkg.crossplane.io provider-aws -o jsonpath="{.status.currentRevision}")
```

### 4. Install configuration package
```
kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-aws:v1.7.0

# Ensure it turns healthy
kubectl get pkg
```
