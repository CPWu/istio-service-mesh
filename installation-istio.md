# Installation Istio

## Summary

The purpose of this section is to cover the different ways you can install Istio on your Kubernetes cluster. We will provide the YAML files that you could easily download to help you in this. 

For the purpose of this section, we will install Istio on a Kuberntes cluster using Docker Desktop.

## Installing Istio

There are multiple ways we can install Istio on a Kubernetes cluster: using Istioctl `istioctl`, using the Istio Operator or using GetMesh CLI. In this module, we will install Istio on a Kubernetes cluster using the Istio operator. In the next section we will also explain how to install Istio using the GetMesh CLI

### Istioctl installation

Istioctl is a command-line tool we can use to install and customize the Istio installation. We generate a YAML file with all Istio resources and then deploy it to the Kubernetes cluster using the command-line tool. 

### Istio Operator

The advantage of Istio operator installation over the `istioctl` is that we don't need to ugprade Istio mannually. Intead, we can deploy the Istio operator that manges the installation for you. We control the operator by updating a custom resource, and the operator applies the configuration changes for you.

### How about production deployments?

There are additional considerations to keep in mind when deciding on a production deployment model of Istio. We can configure Istio to run in different deployment models - potentially spanning multiple clusters and networks and using multiple control planes. We will learn about the other deployment models, multi-cluster installation, and running workloads on virtual machines in the <strong>Advanced Features</strong> module.

### Platform Installation Guides

We can install Istio on different Kubernetes platforms. For the most up-to-date installation instructions for specific cloud vendors, refer to [Platform Setup](https://istio.io/latest/docs/setup/platform-setup/) documentation.

## GetMesh

Istio is one of the most popular and fast-growing open-source projects. Its release schedule can be very agressive for enterprise lifecycle and change management practices. GetMesh's versions of Istio are actively supported for security patches and other bug fixes and have much longer support life than provided by upstream Istio. [GetMesh(https://istio.tetratelabs.io) addresses this concern by testing all Istio versions against different Kubernetes distributions for functional integrity.

Some service mesh customers need to support elevated security requirements. GetMesh addresses compliance by offering two flavors of the Istio distribution.

- tetrate distro tracks the upstream Istio and may have additional patches applied
- tetratefps distro that is a FIPS-compliant version of the tetrate flavor

### How to get started?

The first step is to download GetMesh CLI.  You can install GetMesh on macOS and Linux platforms. We can use the following command to download the latest version of GetMesh and certified Istio:

`curl -sL https://istio.tetratelabs.io/getmesh/install.sh | bash`

We can run the `version` command to ensure GetMesh got successfully installed. For example:

```bash
$ getmesh version
getmesh version: 1.1.1
active istioctl: 1.8.3-tetrate-v0
no running Istio pods in "istio-system"
```

The version command outputs the version of GetMesh, the version of active Istio CLI, and versions of Istio installed on the Kubernetes cluster.

### Installation Istio with GetMesh

GetMesh communicates with the active Kubernetes cluster from the Kubernetes config file.

To install the demo profile of Istio on a currently active Kubernetes cluster, we can use the getmesh istioctl command like this:

```bash
getmesh istioctl install --set profile=demo
```

The command will check the cluster to make sure it's ready to install Istio, and once you confirm, the installer will proceed to install Istio using the selected profile.

If we check the version now, you'll notice that the output shows the control plane and data plane versions.

### Validating the Configuration

The config-validate command allows you to validate the current config and any YAML manifests that are not yet applied. 

The command invokes validations using external sources such as upstream Istio validations, Kiali libraries, and GetMesh custom configuration checks.

Siimilarly, you can also pass in a YAML file to validate it, before deploying it to the cluster. For example:

```bash
$ getmesh config-validate my-resources.yaml
```

### Managing multiple Istio CLIs

We can use the show command to list the currently downloaded versions of Istio:

```bash
getmesh show 
```

Here's how output might look like:

```bash
$ getmesh show
1.10.3-tetrate-v0 (Active)
1.10.3-tetrateflips-v0
1.9.7-tetrate-v0
```

If the version we'd like to use is not available locally on the computer, we can use the getmesh list command to list all trusted Istio versions:

```bash
getmesh list
```

To fetch a specific version, we can use the fetch command:

```bash
getmesh fetch --version 1.8.3 --flavor tetratefips --flavor-version 1
```

When the above command completes, GetMesh sets the fetched Istio CLI version as the active version as the active version of Istio CLI. For example, running the show command now shows the tetratefips version 1.8.3 active

```bash
$ getmesh show
1.8.3-tetratefips-v1 (Active)
```

Similarily, if we run `getmesh istioctl version`, we'll notice the version of Istio CLI that's in use:

```bash
$ getmesh istioctl version

client version: 1.9.0-tetratefips-v1
control plane version: 1.9.5-tetrate-v0
data plane version: 1.9.5-tetrate-v0 (2 proxies)
```

To switch to a different version of the Istio CLI, we can run the `getmesh switch` command:

```bash
getmesh switch --name 1.10.3-tetrate-v0
```

## Discovery Selectors

Discovery selectors were one of the new features introduced in Istio 1.10. Discovery selectors allow us to control which namespaces Istio control plane watches and sends configuration updates for.

By default, the Istio control plane watches and processes updates for all Kubernetes resources in a cluster. All Envoy proxies in the mesh are configured so that they can reach every workload in the mesh and accept traffic on all ports associated with the workloads.

For example, we have deployed two workloads in separate namespaces - foo and bar. Even though we know that foo will never have to talk to bar and the other way around, the endpoints of one service will be included in the list of discovered endpoints of the other service.

If we run istioctl proxy-config command to list all endpoints that the foo workload from the foo namespace can see, you'll notice an entry for the service called bar:

```bash
istioctl proxy-config endpoints deploy/foo.foo
```

If Istio keeps updating the proxies with information about every service in the cluster, even though they're unrelated, we can imagine how this can slow things down. 

If this sounds familiar, you probably know that there's a solution for this already - it's called the Sidecar resource

We'll talk bout the Sidecar resource in a later module.

### Configuring Discovery Selectors

We can configure discovery selectors at the mesh level in the MeshConfig. Discovery selectors are a set of Kubernetes selectors that specify the namespaces Istio watches and updates when pushing configuration to the sidecars.

Like with the Sidecar resource, the discoverySelectors can be used to limit the number of items watched and processed by Istio.

We can update the IstioOperator to include the discoverySelectors field as shown below:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-demo
spec:
  meshConfig: 
    discoverySelectors:
    - matchLabels:
        env: test
```

The above example sets the env=test as a match label. That means the list of namespaces Istio watches and updates will include workloads in namespaces labeled with env=test.

If we label the foo namespace with env=test label and then list the endpoints, we'll notice there are not as many endpoints listed in the configuration now. That's because the only namespace we labeled is the foo namespace, and that's the only namespace the Istio control plane watches and sends updates for.

```bash
istioctl proxy-config endpoints deploy/foo.foo
```

If we label the namespace bar as well and then re-run the `istioctl proxy-config` command, we'll notice the bar endpoint shows up as part of the configuration for the foo service.

## Installation Istio with Operator

In this section, we will install Istio on a Kubernetes cluster using the Istio operator.

### Prerequisites

To install Istio, we will need a running instance of a Kubernetes cluster. All cloud providers have a managed Kubernetes cluster offering we can use to install Istio service mesh.

We can also run a Kubernetes cluster locally on your computer using one of the following platforms:
- Minikube
- Docker Desktop
- kind
- MicroK8s

When using a local Kubernetes cluster, ensure your computer meets the minimum requirements for Istio installation (ex. 16384 MB RAM and 4 CPUs). Also, ensure the Kubernetes cluster version is v1.19.0 or higher.

### Kubernetes CLI

If you need to install the Kubernetes CLI, follow these [instructions](https://kubernetes.io/docs/tasks/tools/).

We can run `kubectl version` to check if the CLI got installed. 

### Download Istio

Throughout this course, we will be using Istio 1.10.3. The first step to installtng istio is downloading the Istio CLI (istioctl), installation manifests, samples and tools.

The easiest way to install the latest version is to use the download Istio script.

Open a terminal window and open the folder where you want to download Istio, then run the download script:

```bash
$ curl -L https://istio.io/downloadIstio | ISTIO_VERISON=1.10.3 sh -
```

Istio release is downloaded and unpacked to the folder called istio-1.10.3 To access istioctl we should add it to the path:

```bash
$ cd istio-1.10-3
$ export PATH=$PWD/bin:$PATH
```

To check istioctl is on the path, run istioctl version. You should see an output like this:

```bash
$ istioctl version
no running Istio pods in "istio-system"
1.10.3
```

### Install Istio

Istio supports multiple configuration profiles. The difference between the profiles is in components that get installed.

```bash
$ istioctl profile list
Istio configuration profiles:
    default
    demo
    empty
    external
    minimal
    openshift
    preview
    remote
```

The recommended profile for production deployments is the `default` profile. We will be install the `demo` profile as it contains all core components, has a high level of tracing and logging enabled, and is meant for learning about different Istio features.

We can also start with the `minimal` component and individually install other features, like ingress and egress gateway, later.

Because we will use the Istio operator for installation, we have to deploy the operator first.

```bash
$ istioctl operator init
Installing operator controller in namespace: istio-operator using image: docker.io/istio/operator:1.11.4
Operator controller will watch namespaces: istio-system
✔Istiooperatorinstalled                                                                                                                                        
✔Installation complete
```

The init command creates the `istio-operator` namespaces and deploys the custom resource definition, operator deployment, and other resources necessary for the operator to work. The operator is ready to use when the installation completes.

To install Istio, we have to create the IstioOperator resource and specfiy the configuration profile we want to use. 

Create a file called `demo-profile.yaml` with the following contents:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: demo-instio-install
spec:
  profile: demo
```

The last thing we need to do is to create the resources:

```bash
$ kubectl apply -f demo-profile.yaml
istiooperator.install.istio.io/demo-istio-install created
```

As soon as the operator detects the Istiooperator resource, it will start installing Istio. The whole process process can take around 5 minutes. 

To check the status of the installation, we can look at the status of the Pods in the `istio-system` namespace.

```bash
$ kubectl get po -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-756d4db566-xs7rr    1/1     Running   0          2m32s
istio-ingressgateway-8577c57fb6-64jl8   1/1     Running   0          2m32s
istiod-6c86784695-n5gb8                 1/1     Running   0          2m39s
```

Or we can check the status of the installation by listing the Istio operator resource. The installation completes once the STATUS column shows HEALTHY:

```bash
$ kubectl get iop -A
NAMESPACE      NAME                 REVISION   STATUS    AGE
istio-system   demo-istio-install              HEALTHY   3m43s
```

The operator has finished installing Istio when all Pods are running, and the operator status is HEALTHY.

### Enable Sidecar Injection

As we've learned in the previous section, service mesh needs the sidecar proxies running alongside each application.

To inject the sidecar proxy into an existing Kubernetes deployment, we can use `kube-inject` action in the istioctl command.

However, we can also enable automatic sidecar injection on any Kubernetes namespace. If we label the namespace with `istio-injection=enabled`, Istio automatically injects the sidecars for any Kubernetes Pods we create in that namespace.

Let's enable automatic sidecar injection on the `default` namespace by adding a label:

```bash
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled
```

To check the namespace is labeled, run the command below. The `default` namespace should be the only one with the value `enabled`.

```bash
$ kubectl get namespace -L istio-injection
NAME              STATUS   AGE   ISTIO-INJECTION
default           Active   22m   enabled
istio-operator    Active   13m   disabled
istio-system      Active   13m   
kube-node-lease   Active   22m   
kube-public       Active   22m   
kube-system       Active   22m  
```

We can now try a Deployment in the `default` namespace and observe the injected proxy. We will create a deployment called `my-nginx` with a single container using image `nginx`:

```bash
$ kubectl create deployment my-nginx --image=nginx
deployment.apps/my-nginx created
```

If we look at the Pods, you will notice there are two containers in the Pod:

```bash
$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-6b74b79f57-vkh4m   2/2     Running   0          73s
```

Similarly, describing the Pod shows Kubernetes created both an `nginx` container and an `istio-proxy` container:

```bash
$ kubectl describe pods my-nginx-6b74b79f57-vkh4m
```

To remove the deployment, run the delete command: 

```
$ kubectl delete deployment my-nginx
deployment.apps "my-nginx" deleted
```

Updating the Operator
To update the operator, we can use kubectl and apply the updated IstioOperator resource. For example, if we wanted to remove the egress gateway, we could update the IstioOperator resource like this:

```bash
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: demo-installation
spec:
  profile: demo
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: false
```

Save the above YAML to iop-egress.yaml and apply it to the cluster using `kubectl apply -f iop-egress.yaml`

If you list the IstioOperator resource, you'll notice the status has changed to RECONCILING, and once the operator removes the egress gateway, the status changes back to HEALTHY.

Another option for updating the Istio installation is to create a seperate IstioOperator resource. That way, you can have a resource for the base installation and separately apply different operators using an empty installation prpfile. For example, here's how you could create a separate IstioOperator resource that only deploys an internal egress gateway:

```bash
apiVersion: install.istop.io/v1alpha1
kind: IstioOperator
metadata:
  name: internal-gateway-only
  namespace: istio-system
spec:
  profile: empty
  components:
    ingressGateways:
      - namespace: some-namespace
        name: ilb-gateway
        enabled: true
        label:
          istio: ilb-gateway
        k8s:
          serviceAnnotations:
            networking.gke.io/load-balancer-type: "Internal"
```

### Updating and Uninstalling Istio

If we want to update the current installation or change the configuration profile, we will need to update the IstioOperator resource deployed earlier. 

To remove the installation, we have to delete the `IstioOperator`, for example:

```bash
$ kubectl delete istiooperator -n istio-system demo-istio-install
```

Once the operator deletes Istio, we can also remove the operator by running:

```bash
$ istopctl operator remove
```

Make sure to delete the IstioOperator resource first before deleting the operator. Otherwise, there might be leftover Istio resources.