# Chapter 09 - Installing Istio

The commands given in the Chapter 09 are given here for copy and paste if you are reading the printed book.

## Make sure that DNS is working
```
$ dig +search +noall +answer ibm.com
```

## Check Istio releases
```
$ curl -L -s https://api.github.com/repos/istio/istio/releases |
grep tag_name
```

## Find out latest version
```
$ export ISTIO_VERSION=$(curl -L -s \
https://api.github.com/repos/istio/istio/releases/latest | grep tag_name | sed "s/ *\"tag_name\": *\"\\(.*\\)\",*/\\1/")

$ echo $ISTIO_VERSION
```

## We will work with Istio 1.3.8

```
$ cd ## Switch to your home directory
$ export ISTIO_VERSION=1.3.8
$ curl -L https://git.io/getLatestIstio | sh -
$ cd istio-$ISTIO_VERSION
```

## Edit .bashrc file

```
$ vi ~/.bashrc

export ISTIO_VERSION=1.3.8
if [ -d ~/istio-${ISTIO_VERSION}/bin ] ; then
export PATH="~/istio-${ISTIO_VERSION}/bin:$PATH"
fi
```

## Source .bashrc file

```
$ source ~/.bashrc
```

## Verify Istio Install

```
$ istioctl verify-install
```

## Reset Helm

```
$ helm reset --force
```
# Installing Istio using the helm template
If using the ***helm template*** command, we need to make sure that we create a **Custom Resource Definition (CRD)** first:
## Create istio-system namespace

```
$ kubectl create namespace istio-system
```

## Grant the **cluster-admin** role to the default service accounts for the **istio-system**
namespace, which we will use for the Istio installation:

```
$ kubectl create clusterrolebinding istio-system-cluster-rolebinding  \
   --clusterrole=cluster-admin --serviceaccount=istiosystem:default
```

## Install the Istio CRDs and re-initialize **tiller** to include the **istio-system** namespace. In the upcoming release for Helm version 3, the dependency
for CRDs will be integrated as a part of the Istio installation either through **helm** or YAML directly:

```
$ cd ~/istio-$ISTIO_VERSION/install/kubernetes/helm/istio-init/files
$ for i in ./crd*yaml; do kubectl apply -f $i; done
```

## Change directory to the Istio helm charts, create a tiller service account, and re-initialize the service:

```
$ cd ~/istio-$ISTIO_VERSION/install/kubernetes/helm
$ kubectl apply -f helm-service-account.yaml
$ helm init --service-account tiller
```

## Validate that the Tiller pod is running in the kube-system namespace and wait for it to become 1/1

```
$ kubectl get pods -n kube-system | grep tiller
```

## Run the following helm template command to generate the yaml file using the default Istio demo configuration parameters defined in values-istiodemo.
yaml. Next, route the generated output to the kubectl apply command.Helm tiller, which is a server-side component of the helm package manager, is
not used in this case

```
$ cd ~/istio-$ISTIO_VERSION
$ helm template install/kubernetes/helm/istio --name istio \
  --namespace istio-system \
  --values install/kubernetes/helm/istio/values-istio-demo.yaml | \
kubectl apply -f -
```

## Apply generated Istio yaml

```
$ cd ~/istio-$ISTIO_VERSION
$ helm template install/kubernetes/helm/istio --name istio \
--namespace istio-system \
--values install/kubernetes/helm/istio/values-istio-demo.yaml | \
kubectl apply -f -
```
Wait for a few minutes as it will start downloading the Docker images from the  public repositories.  
Check the status of the pods in the istio-system namespace.
## When all pods are ready,watch pods (Press ctrl-c when all are ready)

```
$ kubectl -n istio-system get pods --watch
```
The pods with a **Completed** status are one-time jobs. The other pods that are either **1/1** or  **2/2** under the **READY** column and have a **STATUS** of **Running** are Istio's control plane pods.
Now, let's install Istio using our next method, which is by using Helm and Tiller. 

# Install Istio using Helm and Tiller
Since we already installed Istio from the previous step using the helm template, we need to do a proper cleanup of the existing installation:
### Delete existing Istio -Begin to uninstall Istio by generating resource creation scripts and then route them with the kubectl delete command
```
$ cd ~/istio-$ISTIO_VERSION
$ helm template install/kubernetes/helm/istio --name istio \
--namespace istio-system \
--values install/kubernetes/helm/istio/values-istio-demo.yaml |\
kubectl delete -f -
```
The helm install consists of two tasks:  
*  Creating a CRD for Istio and cert-manager
*  Installing Istio using Helm
### First, create the custom resource definitions required by Istio
```
$ cd ~/istio-$ISTIO_VERSION/install/kubernetes/helm
$ helm install ./istio-init --name istio-init --namespace istiosystem
```
### Next, run the helm install command to install istio-demo (permissive mutual TLS) 
```
$ helm install ./istio -f istio/values-istio-demo.yaml \
--name istio --namespace istio-system
```
The output from **helm install** is long. Notice the number of resources deployed, such as Cluster Role, Cluster Role Binding, Config map, Deployment,
Pod, Role, Role Binding, Secret, Service, Service Account, Attribute Manifest,Handler, Instance, Rule, Destination Rule, mutating webhook configuration, Pod
Disruption Budget, and Horizontal Pod Autoscaler.  
The Istio installation can also be accomplished using the **helm template** command to generate the scripts and then routing it to the **kubectl apply** command. Consult the Istio documentation at **https://archive.istio.io/v1.3/docs/setup/install/helm/** for further details.  
After a successful install, you can verify the installation by running **kubectl -n istio-system get pods** and **kubectl -n istio-system get svc**.


### Check deployment resources in istio-system
```
$ kubectl -n istio-system get deployment
```
Make sure that every deployment's number of pods under the **UP-TO-DATE** column matches with those under AVAILABLE.
Next, we will install Istio using a pre-packaged demo profile.
# Installing Istio using a demo profile
The Istio install using direct YAML can be done by using the demo profile. This provides less flexibility compared to Helm, where we could override parameters using --set in the
helm command line. This method is useful in a development environment.  
If you installed Istio using Helm from the previous section, uninstall Istio using helm and tiller using the following commands:
### Delete earlier Helm 

```
$ helm del --purge istio

$ helm del --purge istio-init
```

### Install Istio using a demo profile

```
$ cd ~/istio-$ISTIO_VERSION/
$ kubectl apply -f install/kubernetes/istio-demo.yaml
```
In this section, we have seen three different methods of Istio install. Next, we want to verify whether the installation has been successful or not.

##  Verifying   installation
Verifying our installation is important to ensure that we have got everything right. To verify the installation of Istio, follow these steps:
### Check version of istioctl and the different Istio modules
```
$ istioctl version --short
```

### Check pods The Istio resources are created in the istio-system namespace. Check the
status of the Istio pods
```
$ kubectl -n istio-system get pods
```

## Installing a load balancer

### Make sure that ip_vs is installed and label the node
```
$ sudo lsmod | grep ^ip_vs

$ sudo ipvsadm -ln

$ echo "ip_vs" | sudo tee /etc/modules-load.d/ipvs.conf

$ kubectl label node osc01.servicemesh.local proxy=true

$ helm repo add kaal https://servicemeshbook.github.io/keepalived

$ helm repo update
```

### Update Helm repo, grant cluster admin and install the load balancer

```
$ helm repo update

$ kubectl create clusterrolebinding \
keepalived-cluster-role-binding \
--clusterrole=cluster-admin --serviceaccount=keepalived:default
```

### Install load balancer
```
$ helm install kaal/keepalived --name keepalived \
--namespace keepalived \
--set keepalivedCloudProvider.serviceIPRange="192.168.142.248/29" \
--set nameOverride="lb"
```

### Check pods

```
$ kubectl -n keepalived get pods
```

### Check services

```
$ kubectl -n istio-system get services
```

## Enabling Istio

### For an existing application

```
$ cd ~/servicemesh
$ istioctl kube-inject -f bookinfo.yaml > bookinfo_proxy.yaml
$ cat bookinfo_proxy.yaml
```

### Check the differences in yaml generated

```
$ diff -y bookinfo.yaml bookinfo_proxy.yaml
```

### Deploy bookinfo

```
$ kubectl -n istio-lab apply -f bookinfo_proxy.yaml
```

### Check pods

```
$ kubectl -n istio-lab get pods
```

### For a new application


#### Delete, create and label the namespace

```
$ kubectl delete namespace istio-lab

$ kubectl create namespace istio-lab

$ kubectl label namespace istio-lab istio-injection=enabled
```

### Deploy application
```
$ kubectl -n istio-lab apply -f ~/servicemesh/bookinfo.yaml
```

## Horizontal Pod Scaling

```
$ kubectl -n istio-system get hpa
```

### How to delete HPA

```
$ kubectl -n istio-system delete hpa --all

$ kubectl -n istio-system scale deploy istio-egressgateway --replicas=1
$ kubectl -n istio-system scale deploy istio-ingressgateway --replicas=1
$ kubectl -n istio-system scale deploy istio-pilot --replicas=1
$ kubectl -n istio-system scale deploy istio-policy --replicas=1
$ kubectl -n istio-system scale deploy istio-telemetry --replicas=1
```







