
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh

kubectl create namespace crossplane-system





gcloud auth login

export PROJECT_ID=crossplane-example-$(date +%Y%m%d%H)

gcloud projects create $PROJECT_ID

echo https://console.cloud.google.com/marketplace/product/google/container.googleapis.com?project=$PROJECT_ID



export SA_NAME=crossplane-example-sa

export SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts \
    create $SA_NAME \
    --project $PROJECT_ID

export ROLE=roles/admin

gcloud projects add-iam-policy-binding \
    --role $ROLE $PROJECT_ID \
    --member serviceAccount:$SA

gcloud iam service-accounts keys \
    create creds.json \
    --project $PROJECT_ID \
    --iam-account $SA

kubectl --namespace crossplane-system \
    create secret generic gcp-creds \
    --from-file key=./creds.json





helm repo add crossplane-stable \
    https://charts.crossplane.io/stable

helm repo update

helm upgrade --install \
    crossplane crossplane-stable/crossplane \
    --namespace crossplane-system \
    --create-namespace \
    --wait



kubectl crossplane install provider \
    crossplane/provider-gcp:v0.15.0


kubectl get providers --watch


kubectl apply -f provider.yaml

kubectl apply -f gke.yaml

kubectl get gkeclusters

kubectl get nodepools