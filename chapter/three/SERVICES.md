# Services

Kubernetes Pods are mortal. They are born and when they die, they are not resurrected. ReplicationControllers in particular create and destroy Pods dynamically (e.g. when scaling up or down or when doing rolling updates). While each Pod gets its own IP address, even those IP addresses cannot be relied upon to be stable over time. This leads to a problem: if some set of Pods (let’s call them backends) provides functionality to other Pods (let’s call them frontends) inside the Kubernetes cluster, how do those frontends find out and keep track of which backends are in that set?

Enter `Services`.

> A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service. The set of Pods targeted by a Service is (usually) determined by a Label Selector (see below for why you might want a Service without a selector).

# Defining a service

A Service in Kubernetes is a REST object, similar to a Pod. Like all of the REST objects, a Service definition can be POSTed to the apiserver to create a new instance. For example, suppose you have a set of Pods that each expose port 9376 and carry a label "app=MyApp".
```yml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```
This specification will create a new Service object named “my-service” which targets TCP port 9376 on any Pod with the "app=MyApp" label. This Service will also be assigned an IP address (sometimes called the “cluster IP”), which is used by the service proxies (see below). The Service’s selector will be evaluated continuously and the results will be POSTed to an Endpoints object also named “my-service”.

The Service’s selector will be evaluated continuously and the results will be POSTed to an Endpoints object also named “my-service”

> Note that a Service can map an incoming port to any `targetPort`. By default the `targetPort` will be set to the same value as the `port` field. Kubernetes Services support TCP and UDP for protocols. The default is TCP

## Services without selectors

Services generally abstract access to Kubernetes Pods, but they can also abstract other kinds of backends. For example:

- You want to have an external database cluster in production, but in test you use your own databases.
- You want to point your service to a service in another Namespace or on another cluster.
- You are migrating your workload to Kubernetes and some of your backends run outside of Kubernetes.

In any of these scenarios you can define a service without a selector:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

Because this service has no selector, the corresponding Endpoints object will not be created. You can manually map the service to your own specific endpoints:
```yaml
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```
NOTE: Endpoint IPs may not be loopback (127.0.0.0/8), link-local (169.254.0.0/16), or link-local multicast (224.0.0.0/24).

Accessing a Service without a selector works the same as if it had a selector. The traffic will be routed to endpoints defined by the user (1.2.3.4:9376 in this example).

An ExternalName service is a special case of service that does not have selectors. It does not define any ports or Endpoints. Rather, it serves as a way to return an alias to an external service residing outside the cluster.
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```
When looking up the host my-service.prod.svc.CLUSTER, the cluster DNS service will return a CNAME record with the value my.database.example.com. Accessing such a service works in the same way as others, with the only difference that the redirection happens at the DNS level and no proxying or forwarding occurs. Should you later decide to move your database into your cluster, you can start its pods, add appropriate selectors or endpoints and change the service type

# Virtual IPs and service proxies

Every node in a Kubernetes cluster runs a `kube-proxy`. `kube-proxy` is responsible for implementing a form of virtual IP for Services of type other than `ExternalName`. In Kubernetes v1.0, Services are a “layer 4” (TCP/UDP over IP) construct, the proxy was purely in userspace. In Kubernetes v1.1, the Ingress API was added (beta) to represent “layer 7”(HTTP) services, iptables proxy was added too, and become the default operating mode since Kubernetes v1.2. In Kubernetes v1.8.0-beta.0, ipvs proxy was added.

## Proxy-mode: userspace

In this mode, kube-proxy watches the Kubernetes master for the addition and removal of `Service` and `Endpoints` objects. For each `Service` it opens a port (randomly chosen) on the local node. Any connections to this “proxy port” will be proxied to one of the `Services`'s backend `Pods` (as reported in `Endpoints`). Which backend Pod to use is decided based on the `SessionAffinity` of the Service. Lastly, it installs iptables rules which capture traffic to the Service’s clusterIP (which is virtual) and Port and redirects that traffic to the proxy port which proxies the backend Pod. By default, the choice of backend is round robin.

## Proxy-mode: iptables

In this mode, `kube-proxy` watches the Kubernetes master for the addition and removal of `Service` and `Endpoints` objects. For each `Service`, it installs `iptables` rules which capture traffic to the Service’s `clusterIP` (which is virtual) and `Port` and redirects that traffic to one of the Service’s backend sets. For each `Endpoints` object, it installs iptables rules which select a backend Pod. By default, the choice of backend is random.

Obviously, iptables need not switch back between userspace and kernelspace, it should be faster and more reliable than the userspace proxy. However, unlike the userspace proxier, the iptables proxier cannot automatically retry another Pod if the one it initially selects does not respond, so it depends on having working readiness probes.

## Proxy-mode: ipvs

In this mode, `kube-proxy` watches Kubernetes `Services` and `Endpoints`, calls `netlink` interface to create ipvs rules accordingly and syncs ipvs rules with Kubernetes `Services` and `Endpoints` periodically, to make sure ipvs status is consistent with the expectation. When `Service` is accessed, traffic will be redirected to one of the backend Pods.

Similar to iptables, Ipvs is based on netfilter hook function, but uses hash table as the underlying data structure and works in the kernel space. That means ipvs redirects traffic much faster, and has much better performance when syncing proxy rules. Furthermore, ipvs provides more options for load balancing algorithm, such as:

- `rr`: round-robin
- `lc`: least connection
- `dh`: destination hashing
- `sh`: source hashing
- `se`d: shortest expected delay
- `nq`: never queue

# Multi-Port Services

Many Services need to expose more than one port. For this case, Kubernetes supports multiple port definitions on a Service object. When using multiple ports you must give all of your ports names, so that endpoints can be disambiguated. For example:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  - name: https
    protocol: TCP
    port: 443
    targetPort: 9377
```

# Headless services

Sometimes you don’t need or want load-balancing and a single service IP. In this case, you can create “headless” services by specifying "`None`" for the cluster IP (`spec.clusterIP`).

This option allows developers to reduce coupling to the Kubernetes system by allowing them freedom to do discovery their own way. Applications can still use a self-registration pattern and adapters for other discovery systems could easily be built upon this API.

For such Services, a cluster IP is not allocated, kube-proxy does not handle these services, and there is no load balancing or proxying done by the platform for them. How DNS is automatically configured depends on whether the service has selectors defined.

## With selectors

For headless services that define selectors, the endpoints controller creates Endpoints records in the API, and modifies the DNS configuration to return A records (addresses) that point directly to the Pods backing the Service.

## Without selectors

For headless services that do not define selectors, the endpoints controller does not create Endpoints records. However, the DNS system looks for and configures either:

    CNAME records for ExternalName-type services.
    A records for any Endpoints that share a name with the service, for all other types

# Publishing services - service types

For some parts of your application (e.g. frontends) you may want to expose a `Service` onto an external (outside of your cluster) IP address.

Kubernetes ServiceTypes allow you to specify what kind of service you want. The default is ClusterIP.

Type values and their behaviors are:
- `ClusterIP`: Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster. This is the default ServiceType.
- NodePort: Exposes the service on each Node’s IP at a static port (the NodePort). A ClusterIP service, to which the NodePort service will route, is automatically created. You’ll be able to contact the NodePort service, from outside the cluster, by requesting <NodeIP>:<NodePort>.
- LoadBalancer: Exposes the service externally using a cloud provider’s load balancer. NodePort and ClusterIP services, to which the external load balancer will route, are automatically created.
- ExternalName: Maps the service to the contents of the externalName field (e.g. foo.bar.example.com), by returning a CNAME record with its value. No proxying of any kind is set up. This requires version 1.7 or higher of kube-dns.

### NodePort

If you set the type field to "`NodePort`", the Kubernetes master will allocate a port from a flag-configured range (default: 30000-32767), and each Node will proxy that port (the same port number on every Node) into your Service. That port will be reported in your Service’s spec.ports[*].nodePort field.

If you want a specific port number, you can specify a value in the nodePort field, and the system will allocate you that port or else the API transaction will fail (i.e. you need to take care about possible port collisions yourself). The value you specify must be in the configured range for node ports.

This gives developers the freedom to set up their own load balancers, to configure environments that are not fully supported by Kubernetes, or even to just expose one or more nodes’ IPs directly.

Note that this Service will be visible as both <NodeIP>:spec.ports[*].nodePort and spec.clusterIP:spec.ports[*].port.

### LoadBalancer

On cloud providers which support external load balancers, setting the type field to "LoadBalancer" will provision a load balancer for your Service. The actual creation of the load balancer happens asynchronously, and information about the provisioned balancer will be published in the Service’s status.loadBalancer field. For example:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
  clusterIP: 10.0.171.239
  loadBalancerIP: 78.11.24.19
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 146.148.47.155
```

### Internal load balancer

In a mixed environment it is sometimes necessary to route traffic from services inside the same VPC.

In a split-horizon DNS environment you would need two services to be able to route both external and internal traffic to your endpoints.

This can be achieved by adding the following annotations:

```yaml
[...]
metadata:
    name: my-service
    annotations:
        service.beta.kubernetes.io/aws-load-balancer-internal: 0.0.0.0/0
[...]
```

