# GCP Setup

* Viewing a list of available zones

```
gcloud compute zones list

```

* Viewing information about a region

```
gcloud compute regions describe us-east1 #[REGION]

```

* set your default compute zone:

```
gcloud config set compute/zone us-east1-d

```

* To see a list of available services for a project

```
gcloud services list --available

```
* To enable particular services

```
gcloud services enable container.googleapis.com

```
## Create a Kubernetes Cluster
You'll use Google Container Engine to create and manage your Kubernetes cluster. Provision the cluster with `gcloud`:

* Create a Compute Engine network for the Kubernetes Engine cluster to connect to and use.

```
gcloud compute networks create jenkins
```

```shell
gcloud container clusters create jenkins-cd \
  --network jenkins --machine-type n1-standard-2 --num-nodes 2 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw,cloud-platform"
```

* Confirm that you can connect to your cluster by running a command to check the number of nodes.

```
kubectl get nodes
```

Once that operation completes download the credentials for your cluster using the [gcloud CLI](https://cloud.google.com/sdk/):
```shell
$ gcloud container clusters get-credentials jenkins-cd
```
## Install Helm

In this lab, you will use Helm to install Jenkins from the Charts repository. Helm is a package manager that makes it easy to configure and deploy Kubernetes applications.  Once you have Jenkins installed, you'll be able to set up your CI/CD pipleline.

* Download and install the helm binary

```shell
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
```

* Unzip the file to your local system:

```shell
tar zxfv helm-v2.9.1-linux-amd64.tar.gz
cp linux-amd64/helm .
```

* Add yourself as a cluster administrator in the cluster's RBAC so that you can give Jenkins permissions in the cluster:
    
```shell
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```

* Grant Tiller, the server side of Helm, the cluster-admin role in your cluster:

```shell
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin \
               --serviceaccount=kube-system:tiller
```

* Initialize Helm. This ensures that the server side of Helm (Tiller) is properly installed in your cluster.

```shell
./helm init --service-account=tiller
./helm update
```

* Ensure Helm is properly installed by running the following command. You should see versions appear for both the server and the client of ```v2.9.1```:

```shell
./helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```
