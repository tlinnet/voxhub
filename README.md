# voxhub
Testing deployment of JupyterHub

See:
[zero-to-jupyterhub.readthedocs.io](https://zero-to-jupyterhub.readthedocs.io/en/latest/)

# File with variables

Create this file, which will keep variables
used in the setup.

Later, we will source the variables to oour current shell.

```bash
source 01_vars.sh
```

# Creating a Kubernetes Cluster at Google

See
[setting-up-kubernetes-on-google-cloud](https://zero-to-jupyterhub.readthedocs.io/en/latest/create-k8s-cluster.html#setting-up-kubernetes-on-google-cloud).

Open [console.cloud.google.com](https://console.cloud.google.com)

Create a project, and save variables from Google Project name

```bash
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

[container.googleapis.com/overview](https://console.cloud.google.com/apis/api/container.googleapis.com/overview)

## Install kubectl

```bash
gcloud components install kubectl
```

## Spin up servers

First get at list of [machine-types](https://cloud.google.com/compute/docs/machine-types). 

Pricing is [here](https://cloud.google.com/compute/pricing#machinetype)

Zones is [explained here](https://cloud.google.com/compute/docs/regions-zones/)

Region **europe-west1** is in St. Ghislain, Belgium

```bash
gcloud compute regions list
gcloud compute zones list

gcloud compute machine-types list | head
gcloud compute machine-types list | grep europe
compute machine-types list | grep europe-west1
gcloud compute machine-types list | grep europe-west1 | grep n1-standard-2


gcloud compute regions list
```

Save variables for Kubernetes engine.
You need to start minimum 1 nodes, but with at least **7.5GB Memory.**

```bash
echo "# The Google Cloud Kubernetes Name" >> 01_vars.sh
echo "G_KUBE_NAME=cluster-1" >> 01_vars.sh
echo "# The Google Cloud Kubernetes Region" >> 01_vars.sh
echo "G_KUBE_REGION=europe-west1" >> 01_vars.sh
echo "# The Google Cloud Kubernetes Zone" >> 01_vars.sh
echo "G_KUBE_ZONE=europe-west1-b" >> 01_vars.sh
echo "# The Google Cloud Kubernetes cluster-version" >> 01_vars.sh
echo "G_KUBE_CLUSTERVERSION=1.8.7-gke.1" >> 01_vars.sh
echo "# The Google Cloud Kubernetes machine-type" >> 01_vars.sh
echo "G_KUBE_MACHINETYPE=n1-standard-2" >> 01_vars.sh
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

See configurations
```bash
gcloud config configurations list
gcloud config configurations describe default
```

Create service account
[serviceaccounts](https://console.cloud.google.com/projectselector/iam-admin/serviceaccounts)

```bash
open https://console.cloud.google.com/iam-admin/serviceaccounts/project?project=${G_PROJECT_ID}
# After download, move it to ssh
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

Or with cheaper [preemptible-vm](https://cloud.google.com/kubernetes-engine/docs/concepts/preemptible-vm)

```bash
gcloud config set container/use_v1_api_client false
gcloud config set project $G_PROJECT_ID

gcloud beta container clusters create $G_KUBE_NAME \
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
# Get email of current service user
G_KUBE_SERVICE_USER=$(gcloud config get-value account)
echo $G_KUBE_SERVICE_USER
echo "# The Google Cloud Kubernetes Service user " >> 01_vars.sh
echo "G_KUBE_SERVICE_USER=$G_KUBE_SERVICE_USER" >> 01_vars.sh
```

First see roles permission in google console. 
```bash
open https://console.cloud.google.com/iam-admin/roles/project?project=${G_PROJECT_ID}
```
Filter by: 'Kubernetes'. Look for 'Kubernetes Engine'.
Open them, search for "container.clusterRoleBindings.create".
The role "Kubernetes Engine Admin" has this rule.

Then add this "Kubernetes Engine Admin" role to $G_KUBE_SERVICE_USER:
```bash
echo $G_KUBE_SERVICE_USER
open https://console.cloud.google.com/iam-admin/iam/project?project=${G_PROJECT_ID}
```

Give your account super-user permissions, allowing you to perform all the actions needed to set up JupyterHub.

```bash
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=admin \
    --user=$G_KUBE_SERVICE_USER
```

# Setup Helm

See [setup-helm](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-helm.html)

Helm, the package manager for Kubernetes, is a useful tool to install, upgrade and manage applications on a Kubernetes cluster. 
We will be using Helm to install and manage JupyterHub on our cluster.

```bash
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > install-helm.bash
bash install-helm.bash --version v2.6.2
```

Set up a ServiceAccount for use by Tiller, the server side component of helm.
```bash
kubectl --namespace kube-system create serviceaccount tiller
# Give the ServiceAccount RBAC full permissions to manage the cluser.
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
#Set up Helm on the cluster. This command only needs to run once per Kubernetes cluster.
helm init --service-account tiller
```

## Verify helm

```bash
helm version
```

## Secure Helm

Ensure that tiller is secure from access inside the cluster:

```bash
kubectl --namespace=kube-system patch deployment tiller-deploy --type=json --patch='[{"op": "add", "path": "/spec/template/spec/containers/0/command", "value": ["/tiller", "--listen=localhost:44134"]}]'
```

# Install JupyterHub!

See [setup-jupyterhub](https://zero-to-jupyterhub.readthedocs.io/en/latest/setup-jupyterhub.html#setup-jupyterhub)

Create a random hex string to use as a security token.
```bash
RANDHEX=`openssl rand -hex 32`
echo $RANDHEX
```

Create config.yaml
```bash
echo "proxy:" > config.yaml
echo "  secretToken: '$RANDHEX'" >> config.yaml
cat config.yaml
```

Let’s add the JupyterHub helm repository to your helm, so you can install JupyterHub from it. <br>
This makes it easy to refer to the JupyterHub chart without having to use a long URL each time.

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
```

Now you can install the chart! <br>
Run this command from the directory that contains the config.yaml file to spin up JupyterHub:

Save variables for Helm. 

```bash
G_KUBE_CURCONT=`kubectl config current-context`
echo $G_KUBE_CURCONT
echo "# The Google Cloud Kubernetes current context " >> 01_vars.sh
echo "G_KUBE_CURCONT=$G_KUBE_CURCONT" >> 01_vars.sh

G_KUBE_NAMESPACE=$G_PROJECT_NAME
echo $G_KUBE_NAMESPACE
echo "# The Google Cloud Kubernetes namespace " >> 01_vars.sh
echo "G_KUBE_NAMESPACE=$G_KUBE_NAMESPACE" >> 01_vars.sh

# The relase name must NOT contain underscores "_"
echo "# The Helm version " >> 01_vars.sh
echo "H_VERSION=v0.6" >> 01_vars.sh
echo "# The Helm release name " >> 01_vars.sh
echo "H_RELEASE=jup-01" >> 01_vars.sh

source 01_vars.sh
```

Create name namespace
```bash
# See first
kubectl get namespaces
# This should be empty
kubectl config view | grep namespace:
# create
kubectl config set-context $G_KUBE_CURCONT --namespace=$G_KUBE_NAMESPACE
kubectl config view | grep namespace:
```

Install 
```bash
helm install jupyterhub/jupyterhub --version=$H_VERSION --name=$H_RELEASE --namespace=$G_KUBE_NAMESPACE -f config.yaml
```

## Verify

You can find if the hub and proxy is ready by doing:
```bash
kubectl --namespace=$G_KUBE_NAMESPACE get pod

# Bug finding
HUB=`kubectl --namespace=$G_KUBE_NAMESPACE get pod | grep "hub-" | cut -d " " -f1`
kubectl describe pods $HUB
PROXY=`kubectl --namespace=$G_KUBE_NAMESPACE get pod | grep "proxy-" | cut -d " " -f1`
kubectl describe pods $PROXY
PREPULL1=`kubectl --namespace=$G_KUBE_NAMESPACE get pod | grep "pre-" | head -n 1 | tail -n 1 | cut -d " " -f1`
kubectl describe pods $PREPULL1
PREPULL2=`kubectl --namespace=$G_KUBE_NAMESPACE get pod | grep "pre-" | head -n 2 | tail -n 1 | cut -d " " -f1`
```

You can find the public IP of the JupyterHub by doing:
```bash
kubectl --namespace=$G_KUBE_NAMESPACE get svc
kubectl --namespace=$G_KUBE_NAMESPACE get svc proxy-public
kubectl --namespace=$G_KUBE_NAMESPACE describe svc proxy-public
```

Get the external IP and open
```bash
EXTIP=`kubectl --namespace=$G_KUBE_NAMESPACE get svc proxy-public | grep "LoadBalancer" | awk '{print $4}'`
open http://$EXTIP
```

JupyterHub is running with a default dummy authenticator so entering any username and password combination will let you enter the hub.

If it is NOT running. Check the pods with **kubectl describe pods x**. Is there any memory errors?

# Turning Off JupyterHub and Computational Resources

See [turn-off](https://zero-to-jupyterhub.readthedocs.io/en/latest/turn-off.html)
```bash
helm delete $H_RELEASE --purge
kubectl delete namespace $G_KUBE_NAMESPACE

# Delete clusters
gcloud container clusters list
gcloud container clusters delete $G_KUBE_NAME --zone=$G_KUBE_ZONE

# Check
gcloud container clusters list
kubectl get nodes
kubectl get services
```

Double check to make sure all the resources are now deleted, since anything you have not deleted will cost you money! <br>
You can check the web console (make sure you are in the right project and account) to verify that everything has been deleted.

At a minimum, check the following under the Hamburger (left top corner) menu:

* Compute Engine -> Disks
* Container Engine -> Container Clusters
* Container Registry -> Images
* Networking -> Network Services -> Load Balancing

# Extend jupyterhub

[extending-jupyterhub](https://zero-to-jupyterhub.readthedocs.io/en/latest/extending-jupyterhub.html)

# Mofify DNS A record (Name -> IP adresse)

You have bought "cooldomain.dk", and want the subdomain  "hub.cooldomain.dk"
to point to the google container cluster.

You need to setup an A-record.

At your domain registrar (for example https://web.gratisdns.dk )

Make an A record: A (Navn -> IP adresse)

* Host: hub.cooldomain.dk
* IP (ipv4): $EXTIP
* TTL: 7200 

Wait 24-48 hours.

# Enable HTTPS

See [security](https://zero-to-jupyterhub.readthedocs.io/en/latest/security.html)

By letsencrypt.

Save variables

```bash
echo "# The webpage hostname " >> 01_vars.sh
echo "H_HOST=hub.cooldomain.dk" >> 01_vars.sh
echo "# The webpage hostname " >> 01_vars.sh
echo "H_CONTACTMAIL=mymail@gmail.com" >> 01_vars.sh

source 01_vars.sh
```

First check current state of https 
```bash
open https://$H_HOST
```

Write to config

```bash
echo "  https:" >> config.yaml
echo "    hosts:" >> config.yaml
echo "      - $H_HOST" >> config.yaml
echo "    letsencrypt:" >> config.yaml
echo "      contactEmail: $H_CONTACTMAIL" >> config.yaml
cat config.yaml
```

Run a helm upgrade:
```bash
# See revision
helm list
helm upgrade $H_RELEASE jupyterhub/jupyterhub --version=$H_VERSION --namespace=$G_KUBE_NAMESPACE -f config.yaml
# Revision should have changed
helm list

# Check pods
kubectl --namespace=$G_KUBE_NAMESPACE get pod
```

Wait for about a minute, now your hub should be HTTPS enabled!
```bash
open https://$H_HOST
```

Set the default target for the proxy, by specifying https.<br>
The normal standards can be seen in "values.yaml", <br>
downloaded from jupyterhub-v0.6.tgz https://jupyterhub.github.io/helm-chart/

Or more info here [jupyterhub/configurable-http-proxy](https://github.com/jupyterhub/configurable-http-proxy)

We are changing from **http** to **https**.

```bash
echo "  chp:" >> config.yaml
echo "    cmd:" >> config.yaml
echo '      - configurable-http-proxy' >> config.yaml
echo '      - --ip=0.0.0.0' >> config.yaml
echo '      - --port=8000' >> config.yaml
echo '      - --api-ip=0.0.0.0' >> config.yaml
echo '      - --api-port=8001' >> config.yaml
echo '      - --default-target=https://$(HUB_SERVICE_HOST):$(HUB_SERVICE_PORT)' >> config.yaml
echo '      - --error-target=https://$(HUB_SERVICE_HOST):$(HUB_SERVICE_PORT)' >> config.yaml
echo '      - --log-level=debug' >> config.yaml

cat config.yaml
```

Run a helm upgrade:
```bash
# See revision
helm list
helm upgrade $H_RELEASE jupyterhub/jupyterhub --version=$H_VERSION --namespace=$G_KUBE_NAMESPACE -f config.yaml
# Revision should have changed
helm list

# Check pods
kubectl --namespace=$G_KUBE_NAMESPACE get pod
```

Try
```bash
open http://$H_HOST
```

# Delete the Kubernetes Dashboard

See here: [delete-the-kubernetes-dashboard](http://zero-to-jupyterhub.readthedocs.io/en/latest/security.html#delete-the-kubernetes-dashboard)

```bash
# First inspect
kubectl --namespace=kube-system get deployment
kubectl --namespace=kube-system get deployment kubernetes-dashboard
# Delete
kubectl --namespace=kube-system delete deployment kubernetes-dashboard
kubectl --namespace=kube-system get deployment
```

# Authentication with bitbucket

See [authentication](http://zero-to-jupyterhub.readthedocs.io/en/latest/authentication.html)

* [List of all methods][https://github.com/jupyterhub/oauthenticator]

For bitbucket, see this [oauth-on-bitbucket-cloud](https://confluence.atlassian.com/bitbucket/oauth-on-bitbucket-cloud-238027431.html)

At bitbucket
* Visit https://bitbucket.org/account
* Click **OAuth**
* Click **Add consumer**
* Name: Some random name
* Description: Testing
* Callback URL: https://${H_HOST}/hub/oauth_callback    (replace $H_HOST with your domain)
* Don't change the tick in "This is a private consumer"
* Permissions: Just ask for: Account-->Email
* Click save
* Click the name, and record "Key"-->H_AUTHCLIENTID and "Secret"-->H_AUTHCLIENTSECRET

Save variables

```bash
echo "# The auth type " >> 01_vars.sh
echo "H_AUTHTYPE=bitbucket" >> 01_vars.sh
echo "# The auth clientId " >> 01_vars.sh
echo "H_AUTHCLIENTID=y0urg1thubc1ient1d" >> 01_vars.sh
echo "# The auth clientSecret " >> 01_vars.sh
echo "H_AUTHCLIENTSECRET=an0ther1ongs3cretstr1ng" >> 01_vars.sh

source 01_vars.sh
```

Write config

```bash
echo "" >> config.yaml
echo "hub:" >> config.yaml
echo "  extraEnv:" >> config.yaml
echo "    OAUTH2_AUTHORIZE_URL: https://bitbucket.org/site/oauth2/authorize" >> config.yaml
echo "    OAUTH2_TOKEN_URL: https://bitbucket.org/site/oauth2/access_token" >> config.yaml
echo "auth:" >> config.yaml
echo "  type: custom" >> config.yaml
echo "  custom:" >> config.yaml
echo "    className: oauthenticator.generic.GenericOAuthenticator" >> config.yaml
echo "    config:" >> config.yaml
echo "      client_id: '$H_AUTHCLIENTID'" >> config.yaml
echo "      client_secret: '$H_AUTHCLIENTSECRET'" >> config.yaml
echo "      token_url: https://bitbucket.org/site/oauth2/access_token" >> config.yaml
echo "      userdata_url: https://bitbucket.org/site/oauth2/userinfo" >> config.yaml
echo "      userdata_method: GET" >> config.yaml
echo "      userdata_params: {'state': 'state'}" >> config.yaml
echo "      username_key: preferred_username" >> config.yaml

cat config.yaml
```

Run a helm upgrade:
```bash
# See revision
helm list
helm upgrade $H_RELEASE jupyterhub/jupyterhub --version=$H_VERSION --namespace=$G_KUBE_NAMESPACE -f config.yaml
# Revision should have changed
helm list

# Check pods
kubectl --namespace=$G_KUBE_NAMESPACE get pod
```

Try
```bash
open http://$H_HOST
```

One should get prompted for login with bitbucket.

Afterwards a "500 : Internal Server Error" will display. 

Let us check the logs

```bash
HUB=`kubectl --namespace=$G_KUBE_NAMESPACE get pod | grep "hub-" | cut -d " " -f1`
kubectl describe pods $HUB
kubectl logs $HUB
```

The logs show problems??? 

We delete **hub** and **auth**, and make with github instead

Update, H_AUTHCLIENTID and H_AUTHCLIENTSECRET in 01_vars.sh
```bash
source 01_vars.sh
```

```bash
echo "" >> config.yaml
echo "auth:" >> config.yaml
echo "  type: github" >> config.yaml
echo "  github:" >> config.yaml
echo "    clientId: '$H_AUTHCLIENTID'" >> config.yaml
echo "    clientSecret: '$H_AUTHCLIENTSECRET'" >> config.yaml
echo "    callbackUrl: 'https://${H_HOST}/hub/oauth_callback'" >> config.yaml

cat config.yaml
```

Run a helm upgrade:
```bash
# See revision
helm list
helm upgrade $H_RELEASE jupyterhub/jupyterhub --version=$H_VERSION --namespace=$G_KUBE_NAMESPACE -f config.yaml
# Revision should have changed
helm list

# Check pods
kubectl --namespace=$G_KUBE_NAMESPACE get pod
```
Try
```bash
open http://$H_HOST
```

This works

## Whitelist bitbucket users

First declare a bash array with users to whitelist

```bash
echo "# The array of users to whitelist " >> 01_vars.sh
echo 'H_WHITE=(user1 user2)' >> 01_vars.sh

source 01_vars.sh
```

Try looping over the array

```bash
for x in ${H_WHITE[@]}; do
    echo $x
done
```

Then write to config

```bash
echo "" >> config.yaml
echo "  whitelist:" >> config.yaml
echo "    users:" >> config.yaml

for x in ${H_WHITE[@]}; do
    echo "      - $x" >> config.yaml
done

cat config.yaml 
```

Run a helm upgrade:
```bash
# See revision
helm list
helm upgrade $H_RELEASE jupyterhub/jupyterhub --version=$H_VERSION --namespace=$G_KUBE_NAMESPACE -f config.yaml
# Revision should have changed
helm list

# Check pods
kubectl --namespace=$G_KUBE_NAMESPACE get pod

# Try
open http://$H_HOST
```