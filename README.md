# GCP Setup

* to change to a different project

```
gcloud config set project [PROJECT_ID]

```
* to enable Compute Engine API

```
gcloud services enable compute.googleapis.com

```

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
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin \
        --user=$(gcloud config get-value account)
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
## Configure and Install Jenkins
You will use a custom [values file](https://github.com/kubernetes/helm/blob/master/docs/chart_template_guide/values_files.md) to add the GCP specific plugin necessary to use service account credentials to reach your Cloud Source Repository.

* Use the Helm CLI to deploy the chart with your configuration set.

```shell
./helm install -n jenkinscicd stable/jenkins -f kubernetes-jenkins-cicd/jenkins/values.yaml --version 0.16.6 --wait
```

* Once that command completes ensure the Jenkins pod goes to the `Running` state and the container is in the `READY` state:

```shell
kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
jenkins-cd-85b6c95746-ng9jh   1/1       Running   0          3m
```

* Run the following command to setup port forwarding to the Jenkins UI from the Cloud Shell

```shell
export POD_NAME=$(kubectl get pods --namespace default -l "component=jenkinscicd-master" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```

* Now, check that the Jenkins Service was created properly:

```shell
kubectl get svc
NAME               CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
jenkins-cd         10.35.249.67   <none>        8080/TCP    3h
jenkins-cd-agent   10.35.248.1    <none>        50000/TCP   3h
kubernetes         10.35.240.1    <none>        443/TCP     9h
```

We are using the [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) so that our builder nodes will be automatically launched as necessary when the Jenkins master requests them.
Upon completion of their work they will automatically be turned down and their resources added back to the clusters resource pool.

Notice that this service exposes ports `8080` and `50000` for any pods that match the `selector`. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster.
Additionally the `jenkins-ui` services is exposed using a ClusterIP so that it is not accessible from outside the cluster.

## Connect to Jenkins

* The Jenkins chart will automatically create an admin password for you. To retrieve it, run:

```shell
printf $(kubectl get secret jenkins-cd -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

* To get to the Jenkins user interface, click on the Web Preview button in cloud shell, then click “Preview on port 8080”:

* You should now be able to log in with username `admin` and your auto generated password.
