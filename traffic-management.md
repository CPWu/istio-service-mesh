# Traffic Management

## Overview

In this module, we will get started with routing traffic between services using Istio service mesh. We will learn how to set up an Ingress resource to allow traffic into our cluster as well as an Egress resource to enable traffic to exit the cluster.

Using the traffic routing, we will learn how to deploy new versions of services and run them next to the released, production version of the service, without disrupting production traffic. With both service versions deployed, we will gradually release (canary release) the new version and start routing a percentage of incoming traffic to the latest version.

### Gateways

As part of the Istio installation, we installed the Istio ingress and egress gateways. Both gateways run an instance of the Envoy proxy, and they operate as load balancers at the edge of the mesh. The ingress gateway receives inbound connections, while the egress gateway receives connections going out of the cluster.

Using the ingress gateway, we can apply route rules to the traffic entering the cluster. We can have a single external IP address that points to the ingress gateway and route traffic to different services within the cluster based on the host header.

We can configure both gateways using a Gateway resource. The Gateway resource describes the exposed ports, protocols, SNI (Server Name Indication) configuration for the load balancer, etc.

Under the covers, the Gateway resource controls how the Envoy proxy listens on the network interface and which certificates it presents.

Here’s an example of a Gateway resource:

```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - dev.example.com
    - test.example.com
```

The above Gateway resources set up the Envoy proxy as a load balancer exposing port 80 for ingress. The gateway configuration gets applied to the Istio ingress gateway proxy, which we deployed to the istio-system namespace and has the label istio: ingressgateway set. With a Gateway resource, we can only configure the load balancer. The hosts field acts as a filter and will let through only traffic destined for dev.example.com and test.example.com.

To control and forward the traffic to an actual Kubernetes service running inside the cluster, we have to configure a VirtualService with specific hostnames (dev.example.com and test.example.com for example) and then attach the Gateway to it.

The Ingress gateway we deployed as part of the demo Istio installation created a Kubernetes service with the LoadBalancer type that gets an external IP assigned to it, for example:

```
$ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)                                                                      AGE
istio-egressgateway    ClusterIP      10.0.146.214   <none>           80/TCP,443/TCP,15443/TCP                                                     7m56s
istio-ingressgateway   LoadBalancer   10.0.98.7      XX.XXX.XXX.XXX   15021:31395/TCP,80:32542/TCP,443:31347/TCP,31400:32663/TCP,15443:31525/TCP   7m56s
istiod                 ClusterIP      10.0.66.251    <none>           15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                                8m6s
```

The way the LoadBalancer Kubernetes service type works depends on how and where we’re running the Kubernetes cluster. For a cloud-managed cluster (GCP, AWS, Azure, etc.), a load balancer resource gets provisioned in your cloud account, and the Kubernetes LoadBalancer service will get an external IP address assigned to it. Suppose we’re using Minikube or Docker Desktop. In that case, the external IP address will either be set to localhost (Docker Desktop) or, if we’re using Minikube, it will remain pending, and we will have to use the minikube tunnel command to get an IP address.

In addition to the ingress gateway, we can also deploy an egress gateway to control and filter traffic that’s leaving our mesh.

We can use the same Gateway resource to configure the egress gateway like we configured the ingress gateway. Using the egress gateway allows us to centralize all outgoing traffic, logging, and authorization.

### Simple Routing

We can use the VirtualService resource for traffic routing within the Istio service mesh. With a VirtualService we can define traffic routing rules and apply them when the client tries to connect to the service. An example of this would be sending a request to dev.example.com that eventually ends up at the target service.

Let’s look at an example of running two versions (v1 and v2) of the customers application in the cluster. We have two Kubernetes deployments, customers-v1 and customers-v2. The Pods belonging to these deployments either have a label version: v1 or a label version: v2 set.

We want to configure the VirtualService to route the traffic to the v1 version of the application. The routing to v1 should happen for 70% of the incoming traffic. The 30% of requests should be sent to the v2 version of the application.

Here’s how the VirtualService resource would look like for the above scenario:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers-route
spec:
  hosts:
  - customers.default.svc.cluster.local
  http:
  - name: customers-v1-routes
    route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v1
      weight: 70
  - name: customers-v2-routes
    route:
    - destination:
        host: customers.default.svc.cluster.local
        subset: v2
      weight: 30
```

Under the hosts field, we define the destination host to which the traffic is being sent. In our case, that’s the customers.default.svc.cluster.local Kubernetes service.

The following field is http, and this field contains an ordered list of route rules for HTTP traffic. The destination refers to a service in the service registry and the destination to which the request will be sent after processing the routing rule. The Istio’s service registry contains all Kubernetes services and any services declared with the ServiceEntry resource.

We are also setting the weight on each of the destinations. The weight equals the proportion of the traffic sent to each of the subsets. The sum of all weight should be 100. If we have a single destination, the weight is assumed to be 100.

With the gateways field, we can also specify the gateway names to which we want to bind this VirtualService. For example:

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: customers-route
spec:
  hosts:
    - customers.default.svc.cluster.local
  gateways:
    - my-gateway
  http:
    ...
```

The above YAML binds the customers-route VirtualService to the gateway named my-gateway. Adding the gateway name to the gateways list in the VirtualService exposes the destination routes through the gateway.

When a VirtualService is attached to a Gateway, only the hosts defined in the Gateway resource will be allowed. The following table explains how the hosts field in a Gateway resource acts as a filter and the hosts field in the VirtualService as a match.

| Gateway Hosts                       | VirtualService Hosts                                                       | Behaviour                                                                                                                                                                                                                                                                                      |
|-------------------------------------|----------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| *                                   | customers.default.svc.cluster.local                                        | Traffic is sent through to the VirtualService as * allows all hosts                                                                                                                                                                                                                            |
| customers.default.svc.cluster.local | customers.default.svc.cluster.local                                        | Traffic is sent through as the hosts match                                                                                                                                                                                                                                                     |
| hello.default.svc.cluster.local     | customers.default.svc.cluster.local                                        | Does not work, hosts don’t match                                                                                                                                                                                                                                                               |
| hello.default.svc.cluster.local     | ["hello.default.svc.cluster.local", "customers.default.svc.cluster.local"] | Only hello.default.svc.cluster.local is allowed. It will never allow customers.default.svc.cluster.local through the gateway. However, this is still a valid configuration as the VirtualService could be attached to a second Gateway that has *.default.svc.cluster.local in its hosts field |

### Subsets and Destination Rule

The destinations also refer to different subsets (or service versions). With subsets, we can identify different variants of our application. In our example, we have two subsets, v1 and v2, which correspond to the two different versions of our customer service. Each subset uses a combination of key/value pairs (labels) to determine which Pods to include. We can declare subsets in a resource type called

DestinationRule.

Here’s how the DestinationRule resource looks like with two subsets defined:

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: customers-destination
spec:
  host: customers.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Let’s look at the traffic policies we can set in the DestinationRule.

#### Traffic Policies in Destination Rule

With the DestinationRule, we can define load balancing configuration, connection pool size, outlier detection, etc., to apply to the traffic after the routing has occurred. We can set the traffic policy settings under the trafficPolicy field. Here are the settings:

- Load balancer settings
- Connection pool settings
- Outlier detection
- Client TLS settings
- Port traffic policy

#### Load Balancer Settings

With the load balancer settings, we can control which load balancer algorithm is used for the destination. Here’s an example of the DestinationRule with the traffic policy that sets the load balancing algorithm for the destination to round-robin:

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: customers-destination
spec:
  host: customers.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

We can also set up hash-based load balancing and provide session affinity based on the HTTP headers, cookies, or other request properties. Here’s a snippet of the traffic policy that sets the hash-based load balancing and uses a cookie called ’location` for affinity:

```
trafficPolicy:
  loadBalancer:
    consistentHash:
      httpCookie:
        name: location
        ttl: 4s
```

#### Connection Pool Settings

These settings can be applied to each host in the upstream service at the TCP and HTTP level, and we can use them to control the volume of connections.

Here’s a snippet that shows how we can set a limit of concurrent requests to the service:

```
spec:
  host: myredissrv.prod.svc.cluster.local
  trafficPolicy:
    connectionPool:
      http:
        http2MaxRequests: 50
```

#### Outlier Detection

Outlier detection is a circuit breaker implementation that tracks the status of each host (Pod) in the upstream service. If a host starts returning 5xx HTTP errors, it gets ejected from the load balancing pool for a predefined time. For the TCP services, Envoy counts connection timeouts or failures as errors.

Here’s an example that sets a limit of 500 concurrent HTTP2 requests (http2MaxRequests), with not more than ten requests per connection (maxRequestsPerConnection) to the service. The upstream hosts (Pods) get scanned every 5 minutes (interval), and if any of them fails ten consecutive times (consecutiveErrors), Envoy will eject it for 10 minutes (baseEjectionTime).

```
trafficPolicy:
  connectionPool:
    http:
      http2MaxRequests: 500
      maxRequestsPerConnection: 10
  outlierDetection:
    consecutiveErrors: 10
    interval: 5m
    baseEjectionTime: 10m
```

#### Client TLS Settings

Contains any TLS related settings for connections to the upstream service. Here’s an example of configuring mutual TLS using the provided certificates:

```
trafficPolicy:
  tls:
    mode: MUTUAL
    clientCertificate: /etc/certs/cert.pem
    privateKey: /etc/certs/key.pem
    caCertificates: /etc/certs/ca.pem
Other supported TLS modes are DISABLE (no TLS connection), SIMPLE (originate a TLS connection the upstream endpoint), and ISTIO_MUTUAL (similar to MUTUAL, which uses Istio’s certificates for mTLS).
```

#### Port Traffic Policy

Using the portLevelSettings field we can apply traffic policies to individual ports. For example:

```
trafficPolicy:
  portLevelSettings:
  - port:
      number: 80
    loadBalancer:
      simple: LEAST_CONN
  - port:
      number: 8000
    loadBalancer:
      simple: ROUND_ROBIN
```

### Resiliency

### Failure Injection

### Advanced Routing

### ServiceEntry

### Sidecar

### Envoy Filter

