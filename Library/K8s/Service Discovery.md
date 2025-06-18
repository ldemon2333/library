# What is Service Discovery?
为什么要有服务发现？

Because servers in such an environment may be short-lived and relying on static IP addresses to connect with these systems is not reliable and may lead to system failures and errors.

You could use a DNS endpoint for the servers. However, if the IP changes, you need to manually update the IP address of the DNS.

A service discovery system is a tool that allows servers/services to register themselves and discover other services.

Let's say you have an application that's deployed on multiple servers and a database cluster consisting of six servers. The application needs to connect to the database in order to store and retrieve data.

In our example, the database cluster nodes would register themselves with the service discovery system (service registry), and the application servers would use the service discovery system (service registry) to find the IP addresses of the database servers.

类似加入一个 service registry 的中间层，完成服务发现。

# Service Registry
It is a **centralized service registry** that maintains information about available services. In most cases, it is a key-value store database. The following image shows the dashboard of a service registry of a service discovery tool called [consul](https://www.consul.io/?ref=devopscube.com).
![[Pasted image 20250610213719.png]]

# Types of Service Discovery
### 1. Client-Side Service Discovery Pattern
In the client-side discovery pattern, the client, for example, an application is responsible for interacting with the service registry to get the service details.

For example, the client queries the service registry to get information about the available backend services.

In this model, the client has to implement the service discovery query logic in the application framework. For example, if an application server wants to get the available database server details, the application should implement the query using the service discovery client library to query the service registry.
![[Pasted image 20250610220124.png]]

### 2. Server Side Service Discovery Pattern
In the server-side discovery pattern, the request goes to a load balancer (server) and the load balancer connects to the service registry to get the IP addresses of the healthy backend servers.

Here the **server component** implements the agent or library to query the service registry. One classic example of this model is Nginx load balancing using service discovery. Here the clients (service consumers) directly talk to the Nginx load balancer and the load balancer queries the service registry to get the list of backend servers to route the traffic.

The following image shows service side service discovery using consul.

![[Pasted image 20250610220332.png]]
>If you are aware kubernetes, the [ingress controller](https://devopscube.com/kubernetes-ingress-tutorial/) implements the service side service discovery.

# Service Discovery Tools
To implement the concept of a service registry you need specific tools. Following are the two common [open-source service discovery tools](https://devopscube.com/open-source-service-discovery/).

1. Hashicorp Consul
2. etcd (primarily used for Kubernetes service discovery)

You can use HTTP REST API, DNS, and gRPC protocol with these service discovery tools.

# Service Discovery Hands-On
If you want to get your hands on service discovery concepts, you can try setting up Nginx reverse proxy configuration using consul.

I have published a detailed hands-on guide for the same. Check out the [Service Discovery Example Using Consul](https://devopscube.com/service-discovery-example/)


