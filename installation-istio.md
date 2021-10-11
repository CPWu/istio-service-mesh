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

## Platform Installation Guides

We can install Istio on different Kubernetes platforms. For the most up-to-date installation instructions for specific cloud vendors, refer to [Platform Setup](https://istio.io/latest/docs/setup/platform-setup/) documentation.