# voxhub
Testing deployment of JupyterHub

See:
[zero-to-jupyterhub.readthedocs.io](https://zero-to-jupyterhub.readthedocs.io/en/latest/)

# File with variables

Create this file, which will keep variables
used in the setup.

Source the variables to your current shell.

```bash
source 01_vars.sh
```

# Creating a Kubernetes Cluster at Google

See
[setting-up-kubernetes-on-google-cloud](https://zero-to-jupyterhub.readthedocs.io/en/latest/create-k8s-cluster.html#setting-up-kubernetes-on-google-cloud).

[open https://console.cloud.google.com](https://console.cloud.google.com)

```bash
open https://console.cloud.google.com
```

Save variables from Google Project name

```bash
echo "# The Google Cloud account" >> 01_vars.sh
echo "G_ACCOUNT=mymail@gmail.com" >> 01_vars.sh
echo "# The Google Cloud Project name" >> 01_vars.sh
echo "G_PROJECT_NAME=Name" >> 01_vars.sh
echo "# The Google Cloud  Project ID" >> 01_vars.sh
echo "G_PROJECT_ID=Name-123456" >> 01_vars.sh
echo "# The Google Cloud  Project Number" >> 01_vars.sh
echo "G_PROJECT_NR=123456789012" >> 01_vars.sh
```

Source the variables to your current shell.

```bash
source 01_vars.sh
echo $G_PROJECT_NAME
```

## Enable the Kubernetes Engine API.

[https://console.cloud.google.com/apis/api/container.googleapis.com/overview][https://console.cloud.google.com/apis/api/container.googleapis.com/overview]

## Install kubectl

```bash
gcloud components install kubectl
```

## Spin up servers

First get at list of [machine-types](https://cloud.google.com/compute/docs/machine-types). 

Pricing is [here](https://cloud.google.com/compute/pricing#machinetype)

Zones is [explained here](https://cloud.google.com/compute/docs/regions-zones/)

europe-west3 is in Frankfurt, Germany	

```bash
gcloud compute regions list
gcloud compute zones list

gcloud compute machine-types list | head
gcloud compute machine-types list | grep europe
compute machine-types list | grep europe-west3
gcloud compute machine-types list | grep europe-west3 | grep n1-standard-1

gcloud compute regions list
```

Save variables for Kubernetes engine

```bash
echo "# The Google Cloud Kubernetes Name" >> 01_vars.sh
echo "G_KUBE_NAME=cluster-1" >> 01_vars.sh
echo "# The Google Cloud Kubernetes Region" >> 01_vars.sh
echo "G_KUBE_REGION=europe-west3" >> 01_vars.sh
echo "# The Google Cloud Kubernetes Zone" >> 01_vars.sh
echo "G_KUBE_ZONE=europe-west3-a" >> 01_vars.sh
echo "# The Google Cloud Kubernetes cluster-version" >> 01_vars.sh
echo "G_KUBE_CLUSTERVERSION=1.8.7-gke.1" >> 01_vars.sh
echo "# The Google Cloud Kubernetes machine-type" >> 01_vars.sh
echo "G_KUBE_MACHINETYPE=n1-standard-1" >> 01_vars.sh
echo "# The Google Cloud Kubernetes number of nodes" >> 01_vars.sh
echo "G_KUBE_NUMNODES=1" >> 01_vars.sh
```

Source the variables to your current shell.
```bash
source 01_vars.sh
echo $G_KUBE_NAME
echo $G_KUBE_ZONE
echo $G_KUBE_CLUSTERVERSION 
echo $G_KUBE_MACHINETYPE
echo $G_KUBE_NUMNODES
```

First check projects
```bash
gcloud projects list
```

See configurations
```bash
gcloud config configurations list
gcloud config configurations describe default
```

Create service account
[serviceaccounts](https://console.cloud.google.com/projectselector/iam-admin/serviceaccounts)

```bash
open https://console.cloud.google.com/iam-admin/serviceaccounts/project?project=${G_PROJECT_ID}
# Move it to ssh
ls -la $HOME/Downloads/${G_PROJECT_NAME}-*.json
mv $HOME/Downloads/${G_PROJECT_NAME}-*.json $HOME/.ssh/
ls -la $HOME/.ssh/${G_PROJECT_NAME}-*.json
```

Create configuration
```bash
# List
gcloud config configurations list
# Create empty
gcloud config configurations create $G_PROJECT_NAME
gcloud config configurations list
# Specify
gcloud auth activate-service-account --key-file $HOME/.ssh/${G_PROJECT_NAME}-*.json
gcloud config set compute/region ${G_KUBE_REGION}
gcloud config set compute/zone ${G_KUBE_ZONE}
# Get setup
gcloud config configurations describe $G_PROJECT_NAME
```

Create Cluster
```bash
gcloud config set project $G_PROJECT_ID

gcloud container clusters create $G_KUBE_NAME \
    --zone=$G_KUBE_ZONE \
    --cluster-version=$G_KUBE_CLUSTERVERSION \
    --machine-type=$G_KUBE_MACHINETYPE \
    --num-nodes=$G_KUBE_NUMNODES \
```

```bash
gcloud container clusters list
```

## Inspect in kupectl

Also see [working-with-multiple-projects-on-gke](https://chengl.com/working-with-multiple-projects-on-gke/)

Get kubectl ready by getting GKE credentials for the project
```bash
gcloud container clusters get-credentials $G_KUBE_NAME --zone ${G_KUBE_ZONE} --project $G_PROJECT_ID
```

```bash
kubectl config get-contexts
kubectl config current-context

kubectl get nodes
kubectl get services
```

See [accessing-the-api](https://kubernetes.io/docs/admin/accessing-the-api/)

```bash
cat $HOME/.kube/config
KUBE_USER=`cat $HOME/.kube/config | grep "user: " | tr -d " " | cut -d ":" -f2`
echo $KUBE_USER

SERVICE_USER=$(gcloud config get-value account)
echo $SERVICE_USER
```

First see roles permission in google console. 
```bash
open https://console.cloud.google.com/iam-admin/roles/project?project=${G_PROJECT_ID}
```
Filter by: 'Kubernetes'. Look for 'Kubernetes Engine'.
Open them, search for "container.clusterRoleBindings.create".
The role "Kubernetes Engine Admin" has this rule.

Then add this "Kubernetes Engine Admin" role to $SERVICE_USER:
```bash
echo $SERVICE_USER
open https://console.cloud.google.com/iam-admin/iam/project?project=${G_PROJECT_ID}
```

Give your account super-user permissions, allowing you to perform all the actions needed to set up JupyterHub.

```bash
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=admin \
    --user=$SERVICE_USER
```

# Setup Helm

See [setup-helm](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-helm.html)

Helm, the package manager for Kubernetes, is a useful tool to install, upgrade and manage applications on a Kubernetes cluster. 
We will be using Helm to install and manage JupyterHub on our cluster.

