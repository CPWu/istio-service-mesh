# Services Mesh and Istio Overview

## Summary

The purpose of this section is to provide an overview of what a service mesh is and how it works. It will also provide an understanding of why we would need a service mesh and introduce Istio and its components. 

The goal of this section is to ensure that we
- Understand the importance of service meshes
- Why you would need one
- Understand from a high-level the Istio Service Mesh

## Introduction

Developers frequently break down cloud-native applications into muliple services that perform specific actions. We might have a service that only deals with the Customers and deals with Orders or Payments. All these services communicate with each other through the network. If a new payment needs to be processed, the request gets sent to the Payment service. If customer data needs to be updated, the request gets sent to the Customer service, and so on.

This type of architecture is called microservices architecture. There are a couple of benefits to this architecture. We have multiple smaller teams working on individual services. These teams have the flexibility to pick their tech stack and languages and usually have the autonomy to deploy and release their services independently. The fuel that's behind making these services work is the network that enables services to communicate. As the number of services grows, so does the communication and network chatter between them. The number of services and teams makes monitoring and managing communication logic reasonably complex. 

Since we also know that networks are unreliable and will fail, the combination of all this makes microservices quite complex to manage and monitor.

## Service Mesh Overview

A service mesh is defined as a dedicated infrastructure layer for managing service-to-service communication to make it manageable, visible and controlled. In some versions of the definition, you might also hear about how a service mesh can make the communication between services safe and reliable. If we had to describe a service mesh with a more straightforward sentence, we would say that the service mesh is all about the communication between services. 

But how does the service mesh help with communication? Let's think about the communication logic and where it usually lives. The communication logic is any code that handles the inbound and outbound requests, the retry logic, timeouts, or even perhaps traffic routing. In most cases, developers build this logic as part of the service. So anytime service A calls service B, the request goes through this communication code logic, which decide how to handle the request.

We mentioned that we might end up with a significant number of services if we go with the microservices approach. How do we deal with the communication logic for all these services?
We would create a shared library that contains this logic and reuse it in multiple places. The shared library approach might work fine, assuming we use the same stack or programming language for all of your services. If we don't, we will have to reimplement the library, and reimplementing something is never a fair use of your tine. We might also use services that don't own the codebase for. In that case, we cannot control the communication logic or monitoring. 

The second issue is misconfiguration. In addition to configuring your application, we also have to maintain the communication logic configuration. If we ever need to tweak or update multiple services simultaneously, we will have to do that for each service separately. 

Service mesh takes this commmunication logic, the retries, timeouts and so on, out of the individual service and moves it into a separate infrastructure layer. The infrastructure layer, in the case of a service mesh, is an array of network proxies. The colletion of the network proxies (one next to each service instance) deals with all communication logic between your services. We call these proxies <string>sidecars</strong> because they live alongside each service.

Previously, if we had Customer service talking to the Payment service directly, now we have a proxy next to Customer service talking to a proxy next to Payment service. The collection of these proxies (the infrastructure layer) forms a network mesh, called a service mesh. The service mesh control plane configure the proxies so taht they intercept all in bound and outbound requests transparently. 

The communication logic separated from the business and application logic allows the developers to focus on the business logic, and services mesh operators focus on the service mesh configuration. 

### Why do I need a service mesh?

A service mesh gives us a consistent way to connect, secure and observe microservices. Every failure, succesful call, retry, or timeout can be captured, visualized, and alerted. We can do these scenarios because the proxies capture the requests and metrics from all communication within the mesh. Additionally, we can make decisions based on the request properties. For example, we can inspect the inbound (or outbound) request and write rules that route all requests with a specific header value to a different service version.

All this information and collected metrics make several scenarios reasonably straightforward to implement. Developers and operators can configure and execute the following scenarios without any code changes to the services.:

- mutual TLS and automatic certificate rotation
- identifying performance and reliability issues using metrics
- visualizing metrics in tools like Grafana; this further allows altering and integrating with PagerDuty, for example
- debugging services and tracing using Jaeger or Zipkin*
- weight-based and request based traffic routing, canary deployments, A/B testing 
- traffic monitoring
- increase service resiliency with timeouts and retries
- chaos testing by injecting failure and delays between services
- circuit breakers for detecting and ejecting unhealthy service instances

* Requires minor code changes to propogate tracing headers between services.

## Introducing Istio

Istio is an open-source implementation of a service mesh. At a high level, Istio supoports the following features: 

1. Traffic Management - Using configuration, we can control the flow of traffic betwen services. Setting up circuit breakers, timeouts, or retries can be done with a simple configuration change.
2. Observability - Istio gives us a better understanding of your services through tracing, monitoring, and logging, and it allows us to detect and fix issues quicikly.
3. Security - Istio can manage authenticationm, authorization, and encryption of the communication at the proxy level. We can enforce policies across services with a quick configuration change.

### Istio Components

Istio service mesh has two pieces: a <string>data plane</strong> and a <strong>control plane.</strong>

When building distributed systens, separating the components into a control plane and a data plane is a common pattern. The components in the data plane are on the request path, while the control plane components help the data plane to do its work.

The data plane in Istio consists of Envoy proxies that control the communication between services. The control plane portion of the mesh is responsible for managing and configuring proxies. 

### Envoy (data plane)

Envoy is a high performance proxy developed in C++. Istio service mesh injects the Envoy proxy as a sidecar container next to your application container. The proxy then intercepts all inbound and outbound traffic for that service. Together, the injected proxies form the data plane of a service mesh.

Envoy proxy is also the only component that interacts with the traffic. In addition to the features mentioned earlier - the load balancing, circuit breakers, fault injection, etc. Envoy also supports a pluggable extension model based on WebAssembly (WASM). This extensibility allows us to enforce custom policies and generate telemetry for the traffic in the mesh.

### Istiod (control plane)

Istiod is the control plane component that provides service discovery, configuration, and certificate management features. Istiod takes the high-level rules written in YAML and converts them into an actionable configuration for Envoy. Then, it propogates this configuration to all sidecars in the mesh.

The pilot component inside Istiod abstracts the platform-specific service discovery mechanisms (Kubernetes, Consul, or VMs) and converts them into a standard format that sidecars can consume. 

 Using the built-in identity and credential management, we can enable strong service-to-service and end-user authentication. With authorization features, we can control who can access your services.

 The portion of the control plane, formerly known as Citadel, acts as a certificate authority and generates certificates that allow secure mutual TLS communication between the proxies in the data plane.
 