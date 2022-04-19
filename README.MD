# Crossplane.io on EKS - quick demo

### 1. Install crosplane.io

```
kubectl create namespace crossplane-system

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

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
