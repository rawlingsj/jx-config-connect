# install notes
```
export CLUSTER_NAME=test-cluster-foo
export PROJECT_ID=${PROJECT_ID?}
export ZONE=europe-west1-b
```

```
gcloud services enable cloudresourcemanager.googleapis.com

gcloud beta container clusters create $CLUSTER_NAME \
 --enable-autoscaling \
 --min-nodes=1 \
 --max-nodes=3 \
 --project=$PROJECT_ID \
 --identity-namespace=$PROJECT_ID.svc.id.goog \
 --region=$ZONE \
 --machine-type=n1-standard-4 \
 --num-nodes=2

gcloud iam service-accounts create cnrm-system

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:cnrm-system@$PROJECT_ID.iam.gserviceaccount.com" \
--role="roles/owner"

gcloud iam service-accounts add-iam-policy-binding \
cnrm-system@$PROJECT_ID.iam.gserviceaccount.com \
--member="serviceAccount:$PROJECT_ID.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
--role="roles/iam.workloadIdentityUser"

sed -i.bak 's/${PROJECT_ID?}/'"$PROJECT_ID"'/' install-bundle-workload-identity/0-cnrm-system.yaml
find jx/ -type f -exec sed -i.bak 's/${PROJECT_ID?}/'"$PROJECT_ID"'/' {} \;
find jx/ -type f -exec sed -i.bak 's/${CLUSTER_NAME?}/'"$CLUSTER_NAME"'/' {} \;

kubectl apply -f install-bundle-workload-identity/

kubectl create namespace jx-resources

kubectl annotate namespace jx-resources cnrm.cloud.google.com/project-id=$PROJECT_ID

jx ns jx-resources

kubectl apply -f jx/
```

# validate

```
kubectl create namespace jx
kubectl create serviceaccount tekton-bot --namespace jx
kubectl annotate serviceaccount tekton-bot iam.gke.io/gcp-service-account=$CLUSTER_NAME-tk@$PROJECT_ID.iam.gserviceaccount.com --namespace jx
kubectl run --rm -it \
  --generator=run-pod/v1 \
  --image google/cloud-sdk:slim \
  --serviceaccount tekton-bot \
  --namespace jx \
  config-connect-test
```
inside the pod run
```
gcloud container clusters list
```

## cleanup

```
kubectl delete namespace jx
```

# Fetch latest config controller resources

```bash
gsutil cp gs://cnrm/latest/release-bundle.tar.gz release-bundle.tar.gz

tar zxvf release-bundle.tar.gz
```