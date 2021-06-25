## Overview ##
This repo demonstrates the capabilities of `crossplane` by implementing a simple example.

Crossplane is a k8s native infrastructure provisioning tool that allows you to define you desirsed infrastructure declarativey uskng kubernetes manefests. 

Crossplane utilises the underlying functionality of kubernetes to define resources in a self-healing fashion. Meaning that if a infrastructure resource is destroyed it will be recreated automatically to match the desired state, much in the same way a pod would be when created as part of a deployment.
Crossplane runs ontop of kubernetes and rherefore you must already have a K8s cluster in order to install and use crossplane.

## Prerequisites ##

* Existing Kuberenes Cluster (or create local cluster: [kind](https://kind.sigs.k8s.io/docs/user/quick-start/))
* [Kubectl](https://kubernetes.io/docs/tasks/tools/) 
* [helm]{https://helm.sh/docs/intro/install/}
* GCP Account 
* GCP [Service Account](https://cloud.google.com/iam/docs/creating-managing-service-accounts) and create [Key file](https://cloud.google.com/iam/docs/creating-managing-service-account-keys)
* gcloud cli


## Steps ##

If you need to create cluster locally

    kind create cluster --image kindest/node:v1.16.9 --wait 5m

### Install crossplane

    kubectl create namespace crossplane-system
    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update
    helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

### Install crossplane cli 

    curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh

### Install GCP Provider
    
    kubectl crossplane install configuration registry.upbound.io/xp/getting-started-with-gcp:v1.2.3
    kubectl get pkg

### Create GCP Account Keyfile

Login to your GCP account

    gcloud auth login

Create new project
    export PROJECT_ID=k8s-crossplane-example
    gcloud projects create $PROJECT_ID
    gcloud config set project $PROJECT_ID 

Create new project and service account
    NEW_SA_NAME=k8s-crossplane-example-sa
    SA="${NEW_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
    gcloud iam service-accounts create $NEW_SA_NAME --project $PROJECT_ID

Enable cloud API
    SERVICE="sqladmin.googleapis.com"
    gcloud services enable $SERVICE --project $PROJECT_ID

Grant access to cloud API
    ROLE="roles/cloudsql.admin"
    gcloud projects add-iam-policy-binding --role="$ROLE" $PROJECT_ID --member "serviceAccount:$SA"

Create service account keyfile
    gcloud iam service-accounts keys create creds.json --project $PROJECT_ID --iam-account $SA

### Create a Provider Secret

    kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=./creds.json

### Claim Your Infrastructure
Apply provider configuration 

    kubectl apply -f provider.yaml

Create a XRC (Composite resource claim) for a postgres instance
    
    kubectl apply -f database.yaml

Check status of your claim (may take a while to become READY)

    kubectl get postgresqlinstance my-db

### Consume Your Infrastructure
Define a `Pod` that will show that we are able to connect to our newly provisioned database

    kubectl apply -f pod.yaml

When the pod is created check logs to check the pod successfully connected to database

    kubectl logs see-db 

