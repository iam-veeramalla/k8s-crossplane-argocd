# GitOpsfication of Cloud Infrastucture

Demonstrate GitOpsification of Cloud Infrastructure using Crossplane and Argo CD. We will AWS as a cloud provider in the demo but there is no
restriction in using Azure and GCP as crossplane support all the 3 major cloud platforms.

## Prerequisites

1. A Kubernetes cluster with at least 6 GB of RAM
2. Permissions to create pods and secrets in the Kubernetes cluster
3. Helm version v3.2.0 or later
4. An AWS account with permissions to create an S3 storage bucket
5. AWS access keys

You can also use any local Kubernetes cluster like minikube, k3s, k3d, kind e.t.c.,. for trying out things at your end.

## Install Crossplane

Install using helm charts

```
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
helm repo update
```

once the repo is created and updated, Install the crossplane using the below command.

```
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
```

The above commands will create a namespace for crossplane and installs all the crossplane resource in that namespace.

Wait until the crossplane pods are up and running.

```
kubectl get pods -n crossplane-system
```

```
NAME                                       READY   STATUS    RESTARTS   AGE
crossplane-6ddfcd8748-h6x42                1/1     Running   0          24m
crossplane-rbac-manager-7f9f44bc76-7qhxd   1/1     Running   0          24m
```

## Install Argo CD

There are multiple ways to install Argo CD such as plain kubernetes manifests, helm chart and operator. While the most efficient way is to install 
Argo CD using operator as it offers lifecycle management, upgrade and many. The easiest way to install is using the plain kubernetes manifests.

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

The above commands will create a namespace for argocd and installs all the Argo CD resource in that namespace.

Wait until the Argo CD pods are up and running.

```
kubectl get pods -n argocd
```

```
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          12m
argocd-applicationset-controller-7cdf6fc77b-ss4cf   1/1     Running   0          12m
argocd-dex-server-7d687968db-xmtf4                  1/1     Running   0          12m
argocd-notifications-controller-79dccb6f68-fxngj    1/1     Running   0          12m
argocd-redis-74c8c9c8c6-jkkvc                       1/1     Running   0          12m
argocd-repo-server-98d6d47cc-qh5j6                  1/1     Running   0          12m
argocd-server-567b595777-vsm4c                      1/1     Running   0          12m
```

Awesome, now we have installed the required tools for the project. Both Argo CD and Crossplane are up and running !!!


## Configure AWS

Install the provider into the Kubernetes cluster with a Kubernetes configuration file.

```
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: upbound-provider-aws
spec:
  package: xpkg.upbound.io/upbound/provider-aws:v0.27.0
EOF
```

**Note:** It may take up to five minutes for the provider to list HEALTHY as True.

```
kubectl get providers

NAME                   INSTALLED   HEALTHY   PACKAGE                                        AGE

upbound-provider-aws   True        True      xpkg.upbound.io/upbound/provider-aws:v0.27.0   12m
```

### Create secret for AWS credentials

Create a **text file** containing the AWS account aws_access_key_id and aws_secret_access_key.

`cat aws-credentials.txt`

```
[default]
aws_access_key_id = <aws_access_key>
aws_secret_access_key = <aws_secret_key>
```

**Note:** If you already have AWS CLI configured on your machine, you can get the above details from ~/.aws/credentials file.

Create a Kubernetes secret with the AWS credentials 

```
kubectl create secret generic aws-secret -n crossplane-system --from-file=creds=./aws-credentials.txt
```

### Create a ProviderConfig

```
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF
```

This attaches the AWS credentials, saved as a Kubernetes secret, as a `secretRef` .

The `spec.credentials.secretRef.name` value is the name of the Kubernetes secret containing the AWS credentials
in the `spec.credentials.secretRef.namespace`.

