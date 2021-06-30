## Overview ##
This repo demonstrates the capabilities of `crossplane` by implementing a simple example.

Crossplane is a k8s native infrastructure provisioning tool that allows you to define you desirsed infrastructure declarativey uskng kubernetes manefests. 

Crossplane utilises the underlying functionality of kubernetes to define resources in a self-healing fashion. Meaning that if a infrastructure resource is destroyed it will be recreated automatically to match the desired state, much in the same way a pod would be when created as part of a deployment.
Crossplane runs ontop of kubernetes and rherefore you must already have a K8s cluster in order to install and use crossplane.

#### Key Terms 
`Managed Resources` Resouces that map directly to provider resources (i.e. GCP CloudSQLInstance)
`Composite Resources (XRs)` A custom resource that is composed of other resources, its schema is user-defined.
`Composite Resource Claims (XRCs)` Declares that an application requires particular kind of infrastructure, as well as specifying how to configure it. An XRC is a namespaced proxy for an XR.

XRs are cluster scoped - they exist outside of any namespace. This allows an XRs to represent infrastructure that might be consumed from several different namespaces. An application team may be limited to one namespace, but stil require infrastructure. This infrastructure is made available to application teams via XRCs 

## Prerequisites ##

* Existing Kuberenes Cluster (or create local cluster: [kind](https://kind.sigs.k8s.io/docs/user/quick-start/))
* [Kubectl](https://kubernetes.io/docs/tasks/tools/) 
* [helm](https://helm.sh/docs/intro/install/)
* [GCP Account](https://cloud.google.com/) 
* [gcloud cli](https://cloud.google.com/sdk/docs/install)
* [dockerhub Account](https://hub.docker.com/)

### Create GCP Service Account
Login to your GCP account

    gcloud auth login

Create new GCP project and service account

    # Create new project
    export PROJECT_ID=crossplane-example-$(date +%Y%m%d%H)
    gcloud projects create $PROJECT_ID
    gcloud config set project $PROJECT_ID 

    # Create service account
    export SA_NAME=crossplane-example-sa
    export SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
    gcloud iam service-accounts \
        create $SA_NAME \
        --project $PROJECT_ID

    # Assign role to service account
    export ROLE=roles/admin
    gcloud projects add-iam-policy-binding \
        --role $ROLE $PROJECT_ID \
        --member serviceAccount:$SA

    # Enable Google API for SQL service
    export SERVICE="sqladmin.googleapis.com"
    gcloud services enable $SERVICE --project $PROJECT_ID

    # Create service account keyfile
    gcloud iam service-accounts keys \
        create creds.json \
        --project $PROJECT_ID \
        --iam-account $SA

## Steps ##

If you need to create cluster locally

    kind create cluster

### Install Crossplane on cluster

    kubectl create namespace crossplane-system
    helm repo add crossplane-stable https://charts.crossplane.io/stable
    helm repo update
    helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

### Install Crossplane CLI

    curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh

### Install GCP Provider
    
    kubectl crossplane install provider crossplane/provider-gcp:v0.15.0
    kubectl get providers --watch

### Create a Provider Secret

    kubectl create secret generic gcp-creds -n crossplane-system --from-file=creds=./creds.json

### Build, push and install your configuration package
Crossplane allows us to create a `Configuration` package, in this package we will create a `Composite Resource Definition (XRD)` to define the schema of our Composite Resourses (XRs) and Composite Resource Claims (XRCs).

We can then specify which Managed Resources (MR) our XR & XRC comprise of by defining a `Composition`.

* `crossplane.yaml` - The `Configuration`.
* `definition.yaml` - The `XRD`.
* `composition.yaml` - The `Composition`.

Note: Crossplane can create a configuration from any directory with a valid `crossplane.yaml` metadata file at its root

Build your configuration package, it should create a `.xpkg` file in you current working directory 

    kubectl crossplane build configuration -f configuration

Push package to a registry of your choice (This example uses Dockerhub)

    REG=<dockerhub_username>
    kubectl crossplane push configuration ${REG}/postgres-sql-instance-configuration-package:v1

Install your configuration package on you cluster 

    kubectl crossplane install configuration ${REG}/postgres-sql-instance-configuration-package:v1


### Claim Your Infrastructure
Update `providers.yaml` to use your newly created `PROJECT_ID`

    sed -i "s/<YOUR_PROJECT_ID>/$PROJECT_ID/g" claim/providers.yaml

Create a XRC (Composite resource claim) for a postgres instance

    kubectl apply -f claim/

Check status of your claim (may take a while to become READY)

    kubectl get postgresqlinstance my-db

### Consume Your Infrastructure
Define a `Pod` that will show that we are able to connect to our newly provisioned database

    kubectl apply -f pod.yaml

When the pod is created check logs to check the pod successfully connected to database

    kubectl logs see-db 

### Clean up

    kubectl delete crossplane --all
    kubectl delete ns crossplane-system

### Useful commands

* `kubectl get claim` get all resources of all claim kinds, like PostgreSQLInstance
* `kubectl get composite` get all resources that are of composite kind, like CompositePostgreSQLInstance.
* `kubectl get managed` get all resources that represent a unit of external infrastructure.
* `kubectl get gcp` get all resources related to `gcp` provider.
* `kubectl get crossplane` get all resources related to Crossplane.
