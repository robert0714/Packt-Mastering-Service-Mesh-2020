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

### Check pods The Istio resources are created in the istio-system namespace. Check the status of the Istio pods
```
$ kubectl -n istio-system get pods
```
The pods showing the status of **Completed** are the ones that ran a job successfully. All other pods should show the **Running** status.  
Note from the preceding output that the Istio control plane consists of three  components. They are as follows:  
*  **Citadel**: istio-citadel provides service-to-service and end-user authentication, with built-in identity and credential management.
*  **Mixer**: Mixer consists of istio-policy, istio-telemetry, and istiogalley.
*  **Pilot**: Pilot is istio-pilot. 

**Istio-ingressgateway** and **istio-egressgateway** are platform-independent inbound and outbound traffic gateways. Prometheus, Kiali, and Grafana are backend services for metering and monitoring.

# Installing a load balancer
Managed Kubernetes services such as Google or IBM Cloud will provide an external load balancer. Since our Kubernetes environment is standalone, we do not have an external load balancer; we install and use **keepalived** as a load balancer.

The keepalived load balancer depends on the **ip_vs** kernel module to be loaded. Follow these steps:

### Make sure that the ip_vs kernel module is loaded
```sh
$ sudo lsmod | grep ^ip_vs
```
### If the preceding does not show any output, load the module
```ssh
$ sudo ipvsadm -ln
```
Run **sudo lsmod | grep ^ip_vs** to make sure that the module is loaded.
### Add ip_vs to the module list so that it is loaded automatically on reboot

```sh
$ echo "ip_vs" | sudo tee /etc/modules-load.d/ipvs.conf
```
### The **keepalived** helm chart requires that the node be labeled as **proxy=true** so that it can deploy the daemon set on this master node:
```ssh
$ kubectl label node osc01.servicemesh.local proxy=true
```

### Install keepalived through a helm chart from https://github.com/servicemeshbook/keepalived
```ssh
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

### Check pods. After creating the preceding helm chart, test the readiness and status of pods in the keepalived namespace

```
$ kubectl -n keepalived get pods
```

### Check services. Once the keepalived load balancer is working, check the status of the Istio services, and you should see that the Istio ingress gateway now has an external IP address assigned

```
$ kubectl -n istio-system get services
```
All services should have **cluster-ip** except **jaeger-agent** and **istio-ingressgateway**.They might show as <pending> initially, and **keepalivd** will provide an IP address from a subnet range that we provided to the helm install command. Note the external IP address assigned by the load balancer to **istio-ingressgateway** is **192.168.142.249** , but this could be different in your case.
   
When no external load balancer is used, the node port of the service or port forwarding can be used to run the application from outside the cluster.

Next, we enable Istio for existing applications by injecting a sidecar proxy—which may result in a very short outage of the application as pods need to restart. We will also learn how to enable new applications to get a sidecar proxy injected automatically.

# Enabling Istio
In the previous chapter, we deployed the *Bookinfo* sample microservice in the **istio-lab** namespace. Run **kubectl -n istio-lab get pods** and notice that each pod is running only one container for every microservice.

## For an existing application
To enable Istio for an existing application, we will use **istioctl** to generate additional artifacts in **bookinfo.yaml**, so the sidecar proxy is added to every pod.  

### First, generate modified YAML with a sidecar proxy for the Bookinfo application
```ssh
$ cd ~/servicemesh
$ istioctl kube-inject -f bookinfo.yaml > bookinfo_proxy.yaml
$ cat bookinfo_proxy.yaml
```

### Check the differences in yaml generated. Do diff on the original and modified file to see the additions to the YAML file by the istioctl command

```
$ diff -y bookinfo.yaml bookinfo_proxy.yaml
```
The new definition of the sidecar proxy will be added to the YAML file.
### Deploy bookinfo. Deploy the modified **bookinfo_proxy.yaml** file to inject a sidecar proxy into the existing **bookinfo** microservice

```
$ kubectl -n istio-lab apply -f bookinfo_proxy.yaml
```

### Check pods .Wait a few minutes for the existing pods to terminate and for the new pods to be ready. The output should look similar to the following
```ssh
$ kubectl -n istio-lab get pods
```
Notice that each pod has two running containers since a sidecar proxy was added through the modified YAML.
It is possible to select microservices to not have a sidecar by editing the generated YAML through the istioctl command.

## Enabling Istio for new applications
To show how to enable sidecar injection automatically, we will delete the existing **istio-lab** namespace, and we will redeploy **bookinfo** so that Istio is automatically enabled with the proxy for the new application. Follow these steps

### Delete, create and label the namespace
```ssh
$ kubectl delete namespace istio-lab
```
### Now, create the istio-lab namespace again and label it using istio-injection=enabled
```ssh 
$ kubectl create namespace istio-lab

$ kubectl label namespace istio-lab istio-injection=enabled
```
By labeling an **istio-injection=enabled** namespace, the Istio sidecar gets injected automatically when the application is deployed using **kubectl apply** or the **helm** command.
### Deploy application
```ssh
$ kubectl -n istio-lab apply -f ~/servicemesh/bookinfo.yaml
```
Run **kubectl -n istio-lab get pods** and wait for them to be ready. You will notice that each pod has two containers, and one of them is the sidecar.  
Run **istioctl proxy-status**, which provides the sync status from pilot to each proxy in the mesh

Now that Istio has been enabled, we are ready to learn about its capabilities using examples from https://istio.io . In the next section, we will set up horizontal pod scaling for Istio services.

## Setting up horizontal pod scaling
Each component of Istio has the autoscaling value set to **false** for the **demo** profile (using **install/kubernetes/istio-demo.yaml**). You can set **autoscaleEnabled** to **true** for different components in **install/kubernetes/helm/istio/values-istiodemo.yaml** to enable autoscaling. This configuration may work nicely in production environments based on deployed applications where the autoscaling of pods may help to handle increased workloads.  
To get the benefits of autoscaling, we should be careful in selecting the applications since autoscaling applications in high-latency environments can make the situation go from bad to worse in handling the increased workload.  
Pod scaling can be enabled at the time of the Helm installation if the following parameters
are passed to the **helm install** command using the **--set** argument:
```
mixer.policy.autoscaleEnabled=true
mixer.telemetry.autoscaleEnabled=true
mixer.ingress-gateway.autoscaleEnabled=true
mixer.egress-gateway.autoscaleEnabled=true
pilot.autoscaleEnabled=true
```

Follow the steps given here:
#### Let's check the current autoscaling for every Istio component
```ssh
$ kubectl -n istio-system get hpa
```

### How to delete HPA. If pod scaling is enabled, it can be deleted using the following command. In our case, this is not necessary 

```
$ kubectl -n istio-system delete hpa --all
```
### After deleting auto pod scaling, make sure to set replicas to 1. In our case, it is not necessary

```ssh
$ kubectl -n istio-system scale deploy istio-egressgateway --replicas=1
$ kubectl -n istio-system scale deploy istio-ingressgateway --replicas=1
$ kubectl -n istio-system scale deploy istio-pilot --replicas=1
$ kubectl -n istio-system scale deploy istio-policy --replicas=1
$ kubectl -n istio-system scale deploy istio-telemetry --replicas=1
```

The promise of a service mesh architecture using a solution such as Istio is to effect changes without having to modify the existing application. This is a significant shift in which operations engineers can run modern microservices applications without having to change anything in the code





