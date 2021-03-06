---
tags: reliability
---
## Running highly-available applications

Your customers expect your application to be always available, even during the times when you are deploying new updates to improve the system. Architecting your application well makes your users happy, and your applications and services running without disruptions. You can improve the availability of your application by eliminating single points of failure and making it resilient to failures in individual components. 

You can use Kubernetes to operate your applications and run them in a highly-available and resilient fashion. Kubernetes is designed to run distributed applications. Its declarative system ensures that once you’ve set up the application, Kubernetes will continuously try to [match the current state with the desired state](https://kubernetes.io/docs/concepts/architecture/controller/#desired-vs-current).

## Recommendations

### Avoid running singleton Pods

If you run your applications in a single Pod, then your application will be unavailable if that Pod gets terminated. You can make Kubernetes compensate for the termination of a pod by re-creating Pod automatically. Instead of deploying applications in individual pods, prefer creating [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). If a Pod fails or gets terminated, the Deployment controller will create a new pod.

### Run multiple replicas 

Running multiple replicas of your application will make it highly-available. If one replica of your application fails, your application will still function, albeit at reduced capacity until Kubernetes creates another Pod to make up for the loss. Furthermore, you can use the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) to scale replicas automatically based on workload demand. 

### Schedule replicas across nodes

Running multiple replicas won’t be very useful if all the replicas are running on the same node and the node becomes unavailable. Consider using pod anti-affinity to spread replicas of a deployment across multiple worker nodes. 

You can further improve application’s reliability by running it across multiple AZs. 

The manifest below tells Kubernetes scheduler to *prefer* to place pods on separate nodes and AZs. It doesn’t require distinct nodes or AZ because if it did, then Kubernetes will not be able schedule any pods once there is a pod running in each AZ. If your application requires just 3 replicas then you can use `requiredDuringSchedulingIgnoredDuringExecution` for `topologyKey: failure-domain.beta.kubernetes.io/zone` and Kubernetes scheduler will not schedule two pods in the same AZ.

```
piVersion: apps/v1
kind: Deployment
metadata:
  name: spread-host-az
  labels:
    app: web-server
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-server
              topologyKey: failure-domain.beta.kubernetes.io/zone
            weight: 100
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-server
              topologyKey: kubernetes.io/hostname 
            weight: 99
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

Kubernetes 1.18 introduces [pod topology spread constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/) to allow you to spread Pods across AZs automatically. 

### Run Kubernetes Metrics Server

If you want to scale your applications then install Kubernetes [metrics server](https://github.com/kubernetes-sigs/metrics-server). Kubernetes autoscaler add-ons like [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) and [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) need to track metrics of applications to scale them. The metrics-server is responsible for collecting resource metrics from kubelets and serving them in [Metrics API format](https://github.com/kubernetes/metrics). 

The metrics server doesn’t retain any data and its not a monitoring solution. It’s purpose is to expose CPU and memory usage metrics to other systems. If you want to track the state of your application over a period of time then you need a monitoring like Prometheus or Amazon CloudWatch. 

Follow the [EKS documentation](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html) to install  metrics-server. 

## Horizontal Pod Autoscaler (HPA)

You can avoid impact to your customers during periods of high volume traffic by automatically scaling your application. The HPA allows you to track metrics for applications and scale them. HPA reads metrics from Kubernetes metrics API servers and can use a metric to scale applications. 

The HPA is implemented as a control loop in Kubernetes, it periodically queries metrics from APIs that provide resource metrics.

The HPA can retrieve metrics from the following APIs:
1. `metrics.k8s.io` also known as Resource Metrics API — Provides CPU and memory usage for pods
2. `custom.metrics.k8s.io` — Provides metrics from other metric collectors like Prometheus, these metrics are __internal__ to your Kubernetes cluster. 
3. `external.metrics.k8s.io` — Provides metrics that are __external__ to your Kubernetes cluster (E.g., SQS Queue Depth, ELB latency).

One of these three APIs has to provide the metric that you want to use to scale your application. 

### Scaling applications based on custom or external metrics

If you want to scale your applications based on a metric that is not CPU or memory then you need to rely on custom or external metrics. 

[Custom Metrics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md) API servers provide the `custom-metrics.k8s.io` API which the HPA can use to autoscale applications. 

You can use the [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/directxman12/k8s-prometheus-adapter) to collect metrics from Prometheus and use with the HPA. In this case Prometheus adapter will expose Prometheus metrics in [Metrics API format](https://github.com/kubernetes/metrics/blob/master/pkg/apis/metrics/v1alpha1/types.go). A list of all custom metrics implementation can be found in [Kubernetes Documentation](https://github.com/kubernetes/metrics/blob/master/IMPLEMENTATIONS.md#custom-metrics-api). 

Once you deploy the Prometheus Adapter, you can query custom metrics using kubectl.
`kubectl get —raw /apis/custom.metrics.k8s.io/v1beta1/`

[External metrics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/external-metrics-api.md), as the name suggests provide the Horizontal Pod Autoscaler the ability to scale deployments using metrics that are external to Kubernetes cluster. For example, in batch processing workloads, it is common to scale the number of replicas based on the number of jobs in flight in an SQS queue.

## Vertical Pod Autoscaler (VPA)

You may have to operate applications that cannot be horizontally scaled and the only way to scale them is by increasing the amount of resource allocated to the pod. In such situations you can use the [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) to automatically scale or recommend scaling. 

The VPA automatically adjusts the CPU and memory reservations for your pods to help you “right-size” your applications. 

Your application maybe temporarily available if VPA needs to scale it because VPA’s current implementation does not perform in-place adjustments to pods, instead it will restart the pod that needs to be scaled. 

[EKS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html) includes a walkthrough for setting up VPA. 

[Fairwinds Goldilocks](https://github.com/FairwindsOps/goldilocks/) project simplifies implementation of VPA. Goldilocks can provide VPA recommendations and it can optionally auto-scale the Pods.

## Updating applications

You can use Kubernetes to update your application without causing disruption to your customers. You have few options to update your application. 

### Record changes to deployments when performing in-place upgrades

You can use deployments to update a running application. This is typically done by updating the container image. You can use `kubectl` to update a deployment like this:

```bash
kubectl --record deployment.apps/nginx-deployment set image nginx-deployment nginx=nginx:1.16.1
```

The `--record` argument record the changes to the deployment and helps you if you need to perform a rollback. `kubectl rollout history deployment` shows you the recorded changes to deployments in your cluster. You can rollback a change using `kubectl rollout undo deployment <DEPLOYMENT_NAME>`.

By default when you update a deployment that requires a recreation of pods, deployment will do a [rolling upgrade](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/). This means it will only update a portion of the running pods and not all at once. You can control how deployment performs rolling upgrades through `RollingUpdateStrategy` property. 

When performing a *rolling update* of a Deployment you can use the [`Max Unavailable`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable) property to specify maximum number of Pods that can be unavailable during the update. The `Max Surge` property of Deployment allows you to set the maximum number of Pods that can be created over the desired number of Pods.

### Use Canary deployments

You can make deployments significantly safer by performing canary deployments where changes are made to a small percent of application and monitored. If monitoring shows positive results then new version progressively receives more traffic while the traffic to the old version declines. 

Kubernetes offers no native way to perform canary deployments. You can use [Flagger](https://github.com/weaveworks/flagger) with [Istio](https://docs.flagger.app/tutorials/istio-progressive-delivery) or [App Mesh](https://docs.flagger.app/install/flagger-install-on-eks-appmesh).

## Health checks and self-healing

It’s a truism that no software is bug-free but you can use Kubernetes to minimize the impact of software failures and avoid impacting your customers. 

In the past, if an application crashed, someone had to manually remediate the situation by restarting the application. Kubernetes gives you the ability to detect software failures in your applications and automatically heal them. With Kubernetes you can monitor the health of your applications and automatically remove unhealthy instances.  

Kubernetes supports three types of [health-checks](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/):

1. Liveness probe
2. Startup probe (requires Kubernetes 1.16+)
3. Readiness probe

[Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/) is responsible for running all the above-mentioned checks. Kubelet can check the health of the Pods in three ways, it can either run a shell command inside its container, send a HTTP GET request to its container or open a TCP socket on a specified port. 

## Recommendations
### Use Liveness Probe to remove unhealthy pods
You can use the Liveness probe to detect failure in a running service. For example, if you are running a web service that listens on port 80, you can configure a Liveness probe to send a HTTP GET request on Pod’s port 80. Kubelet will periodically send a GET request to the Pod and expect a response, if the Pod responds between 200-399 then kubelet considers that Pod as healthy, otherwise the Pod will be considered unhealthy. If a Pod fails health-checks continously, kubelet will terminate it. 

You can use `initialDelaySeconds` to delay the first probe.

### Use Startup Probe for applications that take longer to start
When your service needs additional time to startup, you can use the Startup Probe to delay the Liveness Probe. Once the Startup Probe succeeds, the Liveness Probe takes over. You can define maximum time Kubernetes should wait for application startup. If after the maximum configured time, the Pod still fails Startup Probes, it will be terminated and a new Pod will be created. 

### Use Readiness Probe to detect partial unavailability 
While Liveness probe is used to detect failure in an application that can only be remediated by terminating the Pod, Readiness Probe can be used to detect situations where the service may be _temporarily_ unavailable. In these situations the service may become temporarily unresponsive however once this operation completes, it is expected to be healthy again.  

For example, an application shouldn't crash because a dependency such as a database isn't ready or available. Instead, the application should keep retrying to connect to the database until it succeeds. When you make sure that your application can reconnect to a dependency such as a database, you can deliver a more robust and resilient service.

You can use the Readiness Probe to detect such behavior and stop sending requests to the Pod until it becomes functional again. *Unlike Liveness Probe, where a failure would result in a recreation of Pod, a failed Readiness Probe would mean that Pod will not receive any traffic from Kubernetes Service*. When the Liveness Probe succeeds, Pod will resume receiving traffic from Service.

---

## Disruptions

A Pod will run indefinitely unless a user stops it or the worker node it runs on fails. Outside of failed health-checks and autoscaling there aren’t many situations where a pod needs to be terminated. Performing Kubernetes cluster upgrades is one such event. When you  upgrade your Kubernetes cluster, after upgrading the control plane, you will upgrade the worker nodes.

The preferred way to upgrade worker node is terminating old worker nodes and creating new ones. Before terminating a worker nodes, you should `drain` it. When a worker node is drained, all its pods are *safely* evicted. Safely is a key word here, when pods on a worker are evicted, they are not sent a `SIGKILL` signal. Instead a `SIGTERM` signal is sent to the main process of each container in the Pods being evicted. After the `SIGTERM` signal is sent, Kubernetes will give the process some time (grace period) before a `SIGKILL` signal is sent. This grace period is 30 seconds by default, you can override the default by using `grace-period` flag in kubectl. 

`kubectl delete pod <pod name> —grace-period=<seconds>`

Draining also respects `PodDisruptionBudgets`

## Recommendations

### Protect critical workload with Pod Disruption Budgets

Pod Disruption Budget or PDB can temporarily halt the eviction process if the number of replicas of an application fall below the declared threshold. The eviction process will continue once the number of available replicas is over the threshold. You can use PDB to declare the `minAvailable` and `maxUnavailable` number of replicas. For example, if you want at least three copies of your service to be available, you can create a PDB. 

```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: my-svc-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: my-svc
```

This tells Kubernetes to halt the eviction process until three or more replicas are available. 

### Practice chaos engineering 
> Chaos Engineering is discipline of experimenting on a distributed system in order to build confidence in the system’s capability to withstand turbulent conditions in production.

Dominik Tornow in his blog post explains that [Kubernetes is declarative system](https://medium.com/@dominik.tornow/the-mechanics-of-kubernetes-ac8112eaa302) where “*the user supplies a representation of the desired state of the system to the system. Then, the system considers the current state and the desired state to determine the sequence of commands to transition from current state to desired state.*” This means Kubernetes always knows the *desired state* and if the system deviates, Kubernetes can (or at least attempt to) restore the state. For example, if a worker node becomes unavailable, Kubernetes will schedule the Pods on another worker node. Similarly, if a `replica` crashes, the [Deployment Contoller](https://kubernetes.io/docs/concepts/architecture/controller/#design) will create a new `replica`. In this way, Kubernetes controllers automatically fix failures. 

Consider testing the resiliency of your cluster by using a Chaos testing tool like [Gremlin](https://www.gremlin.com) that *breaks things on purpose* to detect failures. 

### Use a Service Mesh

You can use a service mesh to improve your application’s resiliency. Service meshes enable service-to-service communication and increase the observability of your microservices network. Most service mesh products work by having a small network proxy run alongside each service that intercepts and inspects the application’s network traffic. You can place your application in a mesh without modifying your application. Using service proxy’s built-in features, you can have it generate network statistics, create access logs, and add HTTP headers to outbound requests for distributed tracing.

A service mesh can help you make your microservices more resilient with features like automatic request retries, timeouts, circuit-breaking, and rate-limiting.

If you operate multiple clusters, you can use a service mesh to enable cross-cluster service-to-service communication.

### Service Meshes
+ [AWS App Mesh](https://aws.amazon.com/app-mesh/)
+ [Istio](https://istio.io)
+ [LinkerD](http://linkerd.io)
+ [Consul](https://www.consul.io)

---

## Observability 

Observability is an umbrella term that includes monitoring, logging, and tracing. Microservices based applications are distributed by nature. Unlike monolithic applications where monitoring a single system is sufficient, in microservices architecture, each component needs to be monitored individually as well as cohesively as one application. You can use cluster-level monitoring, logging, and distributed tracing systems to identify issues in your cluster before they disrupt your customers.

Kubernetes tools for troubleshooting and monitoring are limited. The metrics-server collects resource metrics and stores them in memory but doesn’t persist them. You can view the logs of a Pod using kubectl, but Kubernetes doesn't automatically retain logs. And the implementation of distributed tracing is done either at application code level or using services meshes. 

Kubernetes extensibility shines here, Kubernetes allows you to bring your own preferred centralized monitoring, logging, and tracing solution. 

## Recommendations

### Monitor your applications

The number of metrics you need to monitor in modern applications is growing continuously. It helps if you have an automated way to track your applications so you can focus on solving your customer’s challenges. Cluster-wide monitoring tools like [Prometheus](https://prometheus.io) or [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html) can be the vigilantes of your cluster and workload and provide you signals when, or preferably, before things go wrong. 

Monitoring tools allow you to create alerts that your operations team can subscribe to. Consider rules to activate alarms for events that can, when exacerbated, lead to an outage or impact application performance. 

Sysdig’s post [Best practices for alerting on Kubernetes](https://sysdig.com/blog/alerting-kubernetes/) includes a comprehensive list of components that can impact the availability of your applications.

### Use Prometheus client library to expose application metrics

In addition to monitoring the state of the application and aggregating standard metrics, you can also use the Prometheus client library to expose application-specific custom metrics to improve the application's observability.

### Use a distributed tracing system to identify bottlenecks

A typical modern application has components distributed over the network and its reliability depends on proper functioning of each of the components that make up the application. You can use a distributed tracing solution to understand how requests flows and how systems communicate. 
Traces can show you where bottlenecks exist in your application network and prevent problems that can cause cascading failures. 

You have two options to implement tracing in your applications: you can either implement distributed tracing at the code level by using shared libraries or you can use a service mesh. 

Implementing tracing at the code level can be disadvantageous. In this method, you have to make changes to your code. This is further complicated if you have polyglot applications. You’re also responsible to maintaining yet another library, across your services. 

Service Meshes like [LinkerD](http://linkerd.io), [Istio](http://istio.io), and [AWS App Mesh](https://aws.amazon.com/app-mesh/) can be used to implement distributed tracing in your application. 

Tracing tools like [AWS X-Ray](https://aws.amazon.com/xray/), [Jaeger](https://www.jaegertracing.io) support both shared library and service mesh implementations. 

Consider using a tracing tool that supports both implementations so you will not have to switch tools if you adopt service mesh. 


