This is a demo showing how to use [Google Cloud Build](https://cloud.google.com/cloud-build/) invocation to deploy to Kubernetes.

### Prerequisites

1.  Create a cluster in Google Kubernetes Engine
    ```
    gcloud container clusters create staging --zone us-central1-c
    ```

    NOTE: Update `cloudbuild.yaml` env options if using a cluster with a different name or zone. 

1. Allow traffic on default potential NodePort range
    ```
    gcloud compute firewall-rules create allow-k8s-nodeports --allow tcp:30000-32767
    ```

1. Add Kubernetes Engine IAM role to Cloud Build Service Account

    ```
    gcloud projects add-iam-policy-binding <PROJECT-ID> \ 
    --member serviceAccount:<PROJECT-NUMBER>@cloudbuild.gserviceaccount.com \
    --role roles/container.developer
    ```

    Learn more about the [Cloud Build Service Account](https://cloud.google.com/cloud-build/docs/securing-builds/set-service-account-permissions#what_is_the_service_account), [Kubernetes Engine Permissions](https://cloud.google.com/kubernetes-engine/docs/how-to/iam) and [Granting Roles to Service Accounts](https://cloud.google.com/iam/docs/granting-roles-to-service-accounts#granting_access_to_a_service_account_for_a_resource).

### Running Build

Deploy to the staging cluster in a unique namespace:
```
gcloud builds submit --config cloudbuild.gke.yaml .
```

### When you're done
1. Delete Kubernetes Cluster
    ```
    gcloud container clusters delete staging --zone us-central1-c
    ```
1. Remove GKE permissions from Cloud Build
    ```
    gcloud projects remove-iam-policy-binding <YOUR-PROJECT-ID> \ 
    --member serviceAccount:<YOUR-PROJECT-NUMBER>@cloudbuild.gserviceaccount.com \
    --role roles/container.<developer-or-admin>
    ```
1. Remove firewall rule
    ```
    gcloud compute firewall-rules delete allow-k8s-nodeports
    ```


## Method 3: Deploy to self-destructing VM

Before beginning, update k8s/db.yaml and k8s/web.yaml with your Project ID.

### to do things locally:
```
# run terraform to create a self-destructing VM w/ microk8s
terraform apply -var="project-name=$(gcloud config get-value project 2> /dev/null)" -var="instance-name=test-$(date +%s)" -auto-approve

# get the IP from terraform
echo $(terraform output ip) > .tmp.microk8s_ip

# patch the IP into the kubectl config
# (backup flag is passed for macOS compatibility)
# TODO: use a better regex so this can work repeatedly instead of just once
sed -i.sed-bak "s/CLUSTER_IP/$(< .tmp.microk8s_ip)/" kubeconfig.microk8s

# kubectl can now deploy to microk8s
kubectl apply -f ./k8s --kubeconfig=kubeconfig.microk8s

# see services
kubectl get services --kubeconfig=kubeconfig.microk8s
```

### to do things in Cloud Build:
(prerequisite: terraform builder is built and pushed to GCR in project)
```
gcloud builds submit --config=cloudbuild.vm.yaml .
```



This is not an official Google product.
