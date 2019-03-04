# The Istio on DCOS/Mesos 

Istio is natively supported by Kubernetes but it could also be integrated with other container orchestration platform like Apache Mesos + Marathon or Mesosphere DC/OS.

In this article, I will use Mesosphere DC/OS to deploy Istio components. 

## Architecture

An Istio service mesh is logically split into a **data plane** and a **control plane**.

* The **data plane** is composed of a set of intelligent proxies ([Envoy](https://https://www.envoyproxy.io/)) deployed as sidecars. These proxies mediate and control all network communication between microservices along with Mixer, a general-purpose policy and telemetry hub.

* The **control plane** manages and configures the proxies to route traffic. Additionally, the control plane configures Mixers to enforce policies and collect telemetry.

The following diagram shows the different components that make up each plane:

![Istio Arch](https://istio.io/docs/concepts/what-is-istio/arch.svg)

## Istio Control Plane Deployment

The istio control plane is composed of **etcd/k8s apiserver**, **Istio Pilot**, **Istio Mixers** (istio-policy, istio-telemetry), **Istio Citadel**.

### Etcd and K8s Apiserver Deployment

Istio configuration yamls like Gateway, VirtualService, DestinationRule, ServiceEntry etc. requires k8s apiserver which accepts yamls requests and stores/persists it to **etcd** store. 

The Pilot lists and watch k8s apiserver to reconcile envoy sidecar configuration to align with custermer's requests. 

Following are the steps to deploy **etcd** and **apiserver**.

```
dcos marathon app add istio-install/etcd.json
dcos marathon app add istio-install/apiserver.json
```

### Jaeger Tracing Deployment

Jaeger isn't part of Istio control plane. But we still need it to provide distributed tracing capability. 

```
dcos marathon app add istio-install/jaegertracing.json
```

### Prometheus Deployment
To reap Istio monitoring capability, we could also deploy prometheus as well.

```
dcos marathon app add istio-install/prometheus-mesos.json
```

### Istio Pilot Deployment

**Pilot** provides service discovery for the Envoy sidecars, traffic management capabilities for intelligent routing (e.g., A/B tests, canary deployments, etc.), and resiliency (timeouts, retries, circuit breakers, etc.). 

We had developed a **Mesos plugin** to enable Pilot to discover Marathon service as well as instances the service managed. 

```
dcos marathon app add istio-install/pilot.json
```

### Istio Citadel Deployment

The **Citadel** in istio control plan is responsible to generate certificate for envoy sidecar to communicate over mTLS. The identification embeded in Citadel created certificate is in **Spiffe** format. 

```
dcos marathon app add istio-install/citadel.json
```

### Istio Mixers Deployment

The istio mixers is composed of **istio-policy** and **istio-telemetry**. It provides check and report interface to envoy sidecar for authorization checking and metrics collection etc. 

The istio mixers deployment requires istio control plane CRDs being populated to k8s apiserver. 

Following steps are executed to populate all necessary CRDs. 

1. To apply CRDs yaml to apiserver, you could deploy below helper container which preinstalled with kubeclt and kubectl config file. 

**Note:** Currently the apiserver isn't exposed to internet access.  

```
dcos marathon app add istio-install/kubectl.json
```

```
> dcos task exec -it kubectl.4845b5d9-3bdd-11e9-9fc5-ae4b60730082 bash

> curl -k -sLO https://raw.githubusercontent.com/jianhuabi/istio-mesos-handson/master/yaml/crds.yaml

> kubectl apply -f crds.yaml
customresourcedefinition.apiextensions.k8s.io "checknothings.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "deniers.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "rules.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "adapters.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "instances.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "templates.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "handlers.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "attributemanifests.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "memquotas.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "quotas.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "stdios.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "metrics.config.istio.io" created
customresourcedefinition.apiextensions.k8s.io "prometheuses.config.istio.io" created

```

2. To deploy istio-policy service.

```
dcos marathon pod add istio-install/istio-policy.json
```

3. To deploy istio-telemetry service.

```
dcos marathon pod add istio-install/istio-telemetry.json
```

### Istio Ingress Gateway Deployment

To access application running on istio mesh via browser, we need to deploy **Istio Ingressgateway**

```
dcos marathon pod add istio-install/istio-ingress-gateway.json
```

### Expose Jaeger UI dashboard over Internet access

We will leverage **marathon-lb** to expose Jaeger UI dashboard over internet, so we could browse Jaeger dashboard to explore tracing feature. 


```
dcos package install marathon-lb --yes

> dcos task |grep marathon-lb
marathon-lb  10.0.4.101  root    R    marathon-lb.9a935818-3bf3-11e9-9fc5-ae4b60730082

#To reivew marathon-lb public ip address, execute below dcos cli.

> dcos node ssh --master-proxy --private-ip=10.0.4.101 "curl ifconfig.io"

34.216.181.222

```

Then you should be able to browse Jaeger dashboard vi below URL

```
http://34.216.181.222:10005/
```

![image alt](https://s3-us-west-2.amazonaws.com/github-png/jaeger-ui.png)


## Deploy BookInfo application to validate Istio Mesos deployment

We will deploy **bookinfo** application via marathon. Below is bookinfo application logical architecture. 

![image alt](https://istio.io/docs/examples/bookinfo/noistio.svg)

```
dcos marathon pod add bookinfo-mesos/productpage.json
dcos marathon pod add bookinfo-mesos/details.json
dcos marathon pod add bookinfo-mesos/ratings.json
dcos marathon pod add bookinfo-mesos/reviews.json
dcos marathon pod add bookinfo-mesos/reviews-v2.json
dcos marathon pod add bookinfo-mesos/reviews-v3.json
```


## BookInfo Sample App Installation

### Expose in-cluster application to Internet

To expose in-cluster applicattion to Internet, we shall configure Istio control plane with **Gateway**, **VirtualService**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: productpage-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage-gateway
spec:
  hosts:
  - "*"
  gateways:
  - productpage-gateway
  http:
  - match:
    - uri:
        prefix: /
    - uri:
        prefix: /productpage
    route:
    - destination:
        port:
          number: 9080
        host: productpage.marathon.slave.mesos
EOF
```

To browse **bookinfo productpage**, we need to acquire istio ingress gateway public IP address as well as the gateway's **http hostport**.

Log in to one of DC/OS node and run below command. 

```bash
>curl http://leader.mesos:8080/v2/pods/istio-ingressgateway::status

"containers": [
        {
          "name": "istio-ingressgateway",
          "status": "TASK_RUNNING",
          "statusSince": "2019-03-01T12:34:35.706Z",
          "conditions": [],
          "containerId": "istio-ingressgateway.instance-56c13a3f-3c1e-11e9-9fc5-ae4b60730082.istio-ingressgateway",
          "endpoints": [
            {
            ### this port is what we required to access bookinfo productpage
              "name": "http2",
              "allocatedHostPort": 12240
            },
            {
              "name": "https",
              "allocatedHostPort": 12241
            },
            {
              "name": "tcp",
              "allocatedHostPort": 12242
            },
            {
              "name": "tcp-pilot-grpc-tls",
              "allocatedHostPort": 12243
            },

```

We should be able to access **bookinfo productpage** via below url.

```
http://{istio-ingress-gateway-publicip}:12240/productpage
```

## Isito Traffic Management 

### Routing Incoming traffic to versioned reviews service.

We have deployed **reviews** with 3 different versions that is **v1, v2, v3**. We want to leverage Istio policy to forward all incoming **review traffic** to only **version v3**. 

Before we can use Istio to control the Bookinfo version routing, we need to define the available **versions, called subsets**, in destination rules.. 

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details.marathon.slave.mesos
  subsets:
  - name: v1
    labels:
      version: v1
---

```

To route to one version only, you apply virtual services that set the default version for the microservices. In this case, the virtual services will route all traffic to **v3** of each **reviews** service.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v3
```

### Route based on user identity

Next, you will change the route configuration so that all traffic from a **specific user** is routed to a specific service version. In this case, all traffic from a user named **Jason** will be routed to the service **reviews:v2**.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews.marathon.slave.mesos
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v2
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v3

```

### Fault Injection

To test the Bookinfo application microservices for resiliency, inject a **7s delay** between the **reviews:v2** and **ratings** microservices for user **jason**. This test will uncover a bug that was intentionally introduced into the Bookinfo app.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings.marathon.slave.mesos
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percent: 100
        fixedDelay: 7s
    route:
    - destination:
        host: ratings.marathon.slave.mesos
        subset: v1
  - route:
    - destination:
        host: ratings.marathon.slave.mesos
        subset: v1

```

#### Understanding what just happened

You’ve found a bug. There are hard-coded timeouts in the microservices that have caused the **reviews** service to fail.

The **timeout** between the **productpage** and the **reviews** service is **6 seconds** - coded as 3s + 1 retry for 6s total. The timeout between the reviews and ratings service is hard-coded at 10 seconds. Because of the delay we introduced, the /productpage times out prematurely and throws the error.

##### clean up testing

```bash
kubectl delete virtualservice ratings
```

### Traffic Shifting

This task shows you how to gradually migrate traffic from one version of a microservice to another. For example, you might migrate traffic from an older version to a new version.

A common use case is to **migrate traffic gradually** from **one version of a microservice** to **another**. In Istio, you accomplish this goal by configuring a sequence of rules that route a percentage of traffic to one service or another. In this task, you will send **%50** of traffic to **reviews:v1** and **%50** to **reviews:v3**. Then, you will **complete the migration by sending %100 of traffic to reviews:v3**.

1. Transfer **50%** of the traffic from **reviews:v1** to **reviews:v3** with the following command:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v1
      weight: 50
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v3
      weight: 50

```
***Note***
###### With the current Envoy sidecar implementation, you may need to refresh the /productpage many times –perhaps 15 or more–to see the proper distribution. You can modify the rules to route 90% of the traffic to v3 to see red stars more often. ######

2. Assuming you decide that the reviews:v3 microservice is stable, you can route **100%** of the traffic to **reviews:v3** by applying this virtual service:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews.marathon.slave.mesos
  http:
  - route:
    - destination:
        host: reviews.marathon.slave.mesos
        subset: v3
```

## Security mTLS

### Setup

Our example use two microservcie, **httpbin and sleep**, both running with an Envoy sidecar proxy. 

```bash
dcos marathon pod add httpbin-mesos/httpbin.json
dcos marathon pod add httpbin-mesos/sleep.json
```

1. Verify service connectivity.

```
> dcos task |grep sleep

proxy   10.0.1.91   root    R    sleep.instance-96a37793-3e28-11e9-b30e-ee054f69031a.proxy
sleep   10.0.1.91   root    R    sleep.instance-96a37793-3e28-11e9-b30e-ee054f69031a.sleep
```

```bash
> dcos task exec -it sleep.instance-96a37793-3e28-11e9-b30e-ee054f69031a.proxy bash
> curl http://httpbin.marathon.slave.mesos:8000/headers

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.marathon.slave.mesos:8000",
    "User-Agent": "curl/7.47.0",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "29e89d8fbc0186e9",
    "X-B3-Traceid": "29e89d8fbc0186e9",
    "X-Request-Id": "319791a6-8381-90f7-bfec-7b72a8268899"
  }
}
```

### Globally enabling Istio mutual TLS

To set a **mesh-wide authentication policy** that enables mutual TLS, submit mesh authentication policy like below:

```
cat <<EOF | kubectl apply -f -
apiVersion: "authentication.istio.io/v1alpha1"
kind: "MeshPolicy"
metadata:
  name: "default"
spec:
  peers:
  - mtls: {}
EOF
```

At this point, only the **receiving side** is configured to use mutual TLS. If you run the **curl** command between Istio services (i.e those with sidecars), all requests will fail with a **503 error code** as the client side is still using plain-text.

```bash
> curl http://httpbin.marathon.slave.mesos:8000/headers -v

.....
HTTP/1.1 503 Service Unavailable
.....
```

We could also use **Pilot** debug endpoint **authenticationz** to reveal mTLS conflicts in your mesh config. 

```bash
>  curl http://localhost:15007/debug/authenticationz
>  

{
    "host": "httpbin.marathon.slave.mesos",
    "port": 8000,
    "authentication_policy_name": "default/",
    "destination_rule_name": "-",
    "server_protocol": "mTLS",
    "client_protocol": "HTTP",
    "TLS_conflict_status": "CONFLICT"
  },
```

The debug message tells us that **httpbin microservcie** requires client to use mTLS to setup connection with it, yet the client is configured with HTTP to setup connection. 


To configure the client side, you need to set destination rules to use mutual TLS. 

```bash
cat <<EOF | kubectl apply -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "default"
spec:
  host: "*.marathon.slave.mesos"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

After submit our destinationrule, let's double confirm the configuration is working via below command. 


```bash
>  curl http://localhost:15007/debug/authenticationz
{
    "host": "httpbin.marathon.slave.mesos",
    "port": 8000,
    "authentication_policy_name": "default/",
    "destination_rule_name": "default/default",
    "server_protocol": "mTLS",
    "client_protocol": "mTLS",
    "TLS_conflict_status": "OK"
  },
```

Then you should be able to get response from **httpbin**.

```bash
>curl http://httpbin.marathon.slave.mesos:8000/headers

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.marathon.slave.mesos:8000",
    "User-Agent": "curl/7.47.0",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "2e27424efbc4092a",
    "X-B3-Traceid": "2e27424efbc4092a",
    "X-Request-Id": "94039e8c-ab7c-9a26-b870-f31b6650f31f"
  }
}
```

#### cleanup

```bash
> kubectl delete destinationrule default
> kubectl delete meshpolicy default
```

### Enable mutual TLS service

You can also set authentication policy and destination rule for a specific service. Run this command to set another policy only for **httpbin** service.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin"
spec:
  host: "httpbin.marathon.slave.mesos"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

If you **curl httpbin** service just now, you shall get **503** error messoage. 

```bash
> curl http://httpbin.marathon.slave.mesos:8000/headers -v

.....
HTTP/1.1 503 Service Unavailable
.....

```

And a destination rule:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: "authentication.istio.io/v1alpha1"
kind: "Policy"
metadata:
  name: "httpbin"
spec:
  targets:
  - name: httpbin.marathon.slave.mesos
  peers:
  - mtls: {}
EOF
```

As expected, you shall be able to get response from httpbin again. 

```bash
> curl http://httpbin.marathon.slave.mesos:8000/headers

{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "httpbin.marathon.slave.mesos:8000",
    "User-Agent": "curl/7.47.0",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "d4b065322bfe870b",
    "X-B3-Traceid": "d4b065322bfe870b",
    "X-Request-Id": "5e4d37b9-230d-9f80-93bc-3880e7eed148"
  }
}
```

## Summary

Up to now, there are some limitations regards to Istio on DC/OS or Mesos soluiton and implemetation. 

1. We don't have a helper tools to assit sidecar injection to a marathon service yaml defination.
2. Don't support namespace concepts on Mesos or DC/OS which is a defacto workload islation concepts on K8s. 
3. Just now, the Pilot on Mesos plugin only supports domain name of **marathon.slave.mesos** 
4. SDS API isn't supported on Istio V1.0 

