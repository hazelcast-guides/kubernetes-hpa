Cloud technologies give you on-demand options so that you can create compute, disk or network resources based on your requirements. When your demand changes, you update your infrastructure by releasing some resources or adding more. That is actually named Manual Scaling based on human intervention. Kubernetes is no different in this particular use case. If you create a service on top of Kubernetes and you see unexpected popularity than you planned for, you need to to scale out your number of Pods to match the traffic coming to your application. Kubernetes has an automated solution for this problem. [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)(HPA) automatically scales the number of pods in Kubernetes based on your metrics selection. You have two options to choose.

*   [Resource Metrics](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/#resource-metrics-pipeline)
*   [Custom Metrics](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/#full-metrics-pipeline)

Lets dig into each of them with a supporting example using Hazelcast

# Resource Metrics

## Install Kubernetes Cluster

As you can read the details from official Kubernetes documentation, [Resource Metrics](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/#resource-metrics-pipeline) provide CPU and Memory based metrics for Pods and Nodes in your Kubernetes Cluster. Those metrics are exposed via the **metrics.k8s.io** **API** and one implementation of that API is [Metrics Server](https://github.com/kubernetes-incubator/metrics-server). If you install metrics-server into your cluster, you can start using **kubectl top** or **HPA**. Let's now see how we can autoscale Hazelcast Cluster using Resource Metrics. First, we need to have a kubernetes cluster with metrics-server deployed. I use GKE in this example. You can create a [GCP trial account](https://cloud.google.com/free/) , [install gcloud](https://cloud.google.com/sdk/install) and execute following command to create kubernetes cluster.

    gcloud container clusters create hazelcast-hpa-test-cluster

This will create a kubernetes cluster with your default zone and project settings.

## Install Helm

Helm is the package manager for Kubernetes and we will use it throughout the document to install various software. This is the [link](https://github.com/helm/helm#install) to install helm into your computer. Once you have a working helm cli and if it is helm v2 then you need to install Tiller too by executing each of the following commands.

    $ kubectl create serviceaccount tiller --namespace kube-system
    $ kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    $ helm init --service-account=tiller

To verify helm installation, just check helm version command

    $ helm version
    Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}

## Install Hazelcast Cluster

This will install 3 member Hazelcast Cluster with Management Center.

    helm install --name hazelcast stable/hazelcast

## Horizontal Pod AutoScaler (Resource Metrics)

As explained before, Metrics Server is the provider for Metrics API and this API is used by HPA for Resource Metrics based autoscaling options. Before moving forward, verify that Metrics Server is properly installed and in the list of API Registration.

    $kubectl get apiservices.apiregistration.k8s.io | grep metrics-server

<span class="s2">v1beta1.metrics.k8s.io kube-system/metrics-server <span class="Apple-converted-space"></span> True<span class="Apple-converted-space"></span> 16m</span></pre>

Let's create an HPA based on CPU usage.

<pre>$ kubectl autoscale statefulset hazelcast --cpu-percent=50 --min=3 --max=10
<span class="s1">horizontalpodautoscaler.autoscaling/hazelcast autoscaled</span></pre>

This HPA will periodically check Hazelcast StatefulSet CPU usage and will decide number of running pods between 3 to 10 based on some [calculation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#algorithm-details). Simplest way to put some cpu load on Hazelcast Pod is executing [yes tool](https://en.wikipedia.org/wiki/Yes_(Unix)). This is just to show how HPA is triggered to scale up Hazelcast Cluster by printing yes in one of Hazelcast Pods. You should use a proper load testing tool to test HPA in your Hazelcast Cluster. Before generating CPU load, you can open 2 new terminals to watch HPA target values and number of hazelcast pods via **watch kubernetes get pods** and **watch kubernetes get hpa **commands. Let's move on and execute following command for 5-10 seconds and terminate via **Ctrl + C**

<pre class="p1"><span class="s1">kubectl exec hazelcast-0 yes > /dev/null</span></pre>

You should see now that your HPA target is above 50% and some new pods are started. As initial hazelcast cluster was 3 member cluster, **hazelcast-3** and above are new pods created by HPA.

<pre class="p1"><span class="s1">**$** </span><span class="s2">kubectl get hpa</span> <span class="s2">NAME<span class="Apple-converted-space"></span> REFERENCE <span class="Apple-converted-space"></span> TARGETS <span class="Apple-converted-space"></span> MINPODS <span class="Apple-converted-space"></span> MAXPODS <span class="Apple-converted-space"></span> REPLICAS <span class="Apple-converted-space"></span> AGE</span> <span class="s2">hazelcast <span class="Apple-converted-space"></span> StatefulSet/hazelcast <span class="Apple-converted-space"></span> 94%/50% <span class="Apple-converted-space"></span> 3 <span class="Apple-converted-space"></span> 5 <span class="Apple-converted-space"></span> 5<span class="Apple-converted-space"></span> 18m</span></pre>

<pre class="p1"><span class="s1">**$** </span><span class="s2">kubectl get pods</span>

<span class="s2">NAME<span class="Apple-converted-space"></span> READY <span class="Apple-converted-space"></span> STATUS<span class="Apple-converted-space"></span> RESTARTS <span class="Apple-converted-space"></span> AGE</span>

<span class="s2">hazelcast-0 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 30m</span>

<span class="s2">hazelcast-1 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 30m</span>

<span class="s2">hazelcast-2 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 29m</span>

<span class="s2">hazelcast-3 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 14m</span>

<span class="s2">hazelcast-4 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 13m</span> </pre>

## Cleanup

As you successfully managed to use Resource Metrics with Hazelcast, we can clean up resources used up to that point.

<pre class="p1"><span class="s1">$ helm delete hazelcast --purge</span> <span class="s1">release "hazelcast" deleted</span>

$<span class="s1">kubectl delete hpa hazelcast</span> <span class="s1">horizontalpodautoscaler.autoscaling "hazelcast" deleted</span>
</pre>

# Custom Metrics

In the previous section, we have explained how to use Resource Metrics to autoscale your deployments based on CPU or Memory Metrics. Although that is fine for some architectures, those metrics are Kubernetes Pod or Node level so application level autoscaling is not possible with Resource Metrics. Kubernetes introduced Custom Metrics API in order to fill in this gap. When using Custom Metrics API, each container exposes their own metrics and HPA uses those metrics to make autoscaling decisions. In this example, we will use [Prometheus](https://prometheus.io/) as Metrics Storage and [Prometheus Adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter) as the Custom Metrics API Provider.

## Install Prometheus

<pre>helm install --name prometheus stable/prometheus --namespace monitoring</pre>

## Install Prometheus Adapter

Create a custom hazelcast-values.yaml

<pre>rules:
  default: true
  custom:
  - seriesQuery: '{__name__=~"jvm_memory_bytes_(used|max)",area="heap",kubernetes_name=~"hazelcast.*"}'
    seriesFilters:
    - is: ^jvm_memory_bytes_(used|max)$
    resources: 
      overrides:
        kubernetes_pod_name: {resource: "pod"}
        kubernetes_namespace: {resource: "namespace"}
        kubernetes_name: {resource: "service"}
    name:
      matches: ^jvm_memory_bytes_(used|max)$
      as: "on_heap_ratio"
    metricsQuery: max(jvm_memory_bytes_used{<<.LabelMatchers>>}/jvm_memory_bytes_max{<<.LabelMatchers>>}) by (<<.GroupBy>>)
prometheus:
  url: http://prometheus-server # make sure the url is correct
  port: 80</pre>

This configuration will be passed to the helm chart while deploying Prometheus Adapter but let's just go through each part before doing that. The config basically tells Prometheus Adapter:

*   query only jvm_memory_bytes_used and jvm_memory_bytes_max
*   assign kubernetes_* based labels to resources to be able to query via REST URLs like "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/*/on_heap_ratio"
*   give a new, easier name(on_heap_ratio) to the metric that we expose via custom metrics adapter
*   select max value out of all series provided by all PODs

This example uses "max" function while creating "metricsQuery" but you can basically use some other [aggregation operator](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators) like avg in your own configuration. If you saved the file above, you can create a prometheus adapter based on that configuration.

<pre>helm install --name prometheus-adapter stable/prometheus-adapter -f hazelcast-values.yaml --namespace monitoring</pre>

## Install Metrics Enabled Hazelcast Cluster

Let's install a new 3 members hazelcast cluster with metrics enabled. Each hazelcast member container in this new deployment will expose their own metrics data under **/metrics** endpoint. This endpoint exposes metrics in Prometheus format because each hazelcast container is started with [Prometheus JMX Exporter](https://github.com/prometheus/jmx_exporter). This is a feature provided by [Hazelcast Docker](https://github.com/hazelcast/hazelcast-docker) Image. We also set **resources.limits.memory=512Mi **which sets each hazelcast member JVM max heap size to 128Mi. JVM by default grabs 25% of available memory as max heap size.

<pre>helm install --name hazelcast-metrics-enabled stable/hazelcast --set metrics.enabled=true,resources.limits.memory=512Mi</pre>

Verify that the custom rule we provided to Prometheus Adapter is functioning properly. If you see <span class="s1">"Error from server (NotFound): the server could not find the metric on_heap_ratio for services", you might need to wait some time because Prometheus might not have started scraping Hazelcast specific metrics. </span>

<pre>$ kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/*/on_heap_ratio" |jq .
{
  "kind": "MetricValueList",
  "apiVersion": "custom.metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/%2A/on_heap_ratio"
  },
  "items": [
    {
      "describedObject": {
        "kind": "Service",
        "namespace": "default",
        "name": "hazelcast-metrics-enabled-metrics",
        "apiVersion": "/v1"
      },
      "metricName": "on_heap_ratio",
      "timestamp": "2019-10-02T15:09:04Z",
      "value": "136m"
    }
  ]
}

</pre>

The most important part in this output is "value": "136m". The suffix "m" means milli-unit and it is [kubernetes-style quatities](//github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/walkthrough.md#quantity-values) to define metric values. Milli-unit is equivalent to 1000ths of a unit so 136m is actually referring to 3.3% which means **max** value of **on_heap_ratio** seen so far.

## Horizontal Pod AutoScaler (Custom Metrics)

As we have configured Hazelcast,Prometheus and Prometheus Adapter, let's now create a Horizontal Pod AutoScaler based on **on_heap_ratio** metric. Following HPA configuration tells HPA if  targetValue > 200m then scale up my cluster. 200m as we explained above means actually 20%. You can change that number based on your own use case. Save following HPA into a file named heap-based-hpa.yaml

<pre>apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: heap-based-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: hazelcast-metrics-enabled
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Object
      object:
        target:
          kind: Service
          name: hazelcast-metrics-enabled-metrics
        metricName: on_heap_ratio
        targetValue: 200m</pre>

Apply HPA to your cluster with kubectl

<pre>$ kubectl apply -f heap-based-hpa.yaml</pre>

<pre>horizontalpodautoscaler.autoscaling/heap-based-hpa created</pre>

## Generate some Memory Load for HPA

Let's just have a look **TARGETS** part of HPA output.

<pre class="line number1 index0 alt2">$kubectl get hpa heap-based-hpa
NAME             REFERENCE                               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
heap-based-hpa   StatefulSet/hazelcast-metrics-enabled   136m/200m   3         10        3          94s</pre>

As we see, current HPA Target is 136m/200m so if we increase memory usage just 10%  by adding 10MB into the cluster, HPA should trigger a scale up event. I will use Hazelcast Java Client to put some data into cluster but you can use your own language to implement the same functionality. You can see all Hazelcast supported programming languages [here](https://hazelcast.org/clients-languages/). Let's first port forward from our local machine to be able to connect remote k8s hazelcast member pod.

<pre>kubectl port-forward hazelcast-metrics-enabled-0 5701</pre>

Execute following code snippet to put data into Hazelcast Cluster. // start Hazelcast Client with smartRouting enabled ClientConfig cfg = new ClientConfig(); cfg.getNetworkConfig().setSmartRouting(false); HazelcastInstance client = HazelcastClient.newHazelcastClient(cfg); // create Hazelcast Distributed Map "numbers" IMap<Object, Object> numbers = client.getMap("numbers"); // put 10000*1K = 10M to "numbers" int i=0; while (i++ < 10000) numbers.put(i,new byte[1024]); // check the size of "numbers" System.out.println("size:"+numbers.size()); //clean up client.shutdown(); When you start putting data into your hazelcast cluster, you will see that new pods will be created and added to Hazelcast Cluster.

<pre class="p1"><span class="s1">**$** </span><span class="s2">kubectl get hpa heap-based-hpa</span> <span class="s2">NAME <span class="Apple-converted-space"></span> REFERENCE <span class="Apple-converted-space"></span> TARGETS <span class="Apple-converted-space"></span> MINPODS <span class="Apple-converted-space"></span> MAXPODS <span class="Apple-converted-space"></span> REPLICAS <span class="Apple-converted-space"></span> AGE</span> <span class="s2">heap-based-hpa <span class="Apple-converted-space"></span> StatefulSet/hazelcast-metrics-enabled <span class="Apple-converted-space"></span> 247m/200m <span class="Apple-converted-space"></span> 3 <span class="Apple-converted-space"></span> 10<span class="Apple-converted-space"></span> 4<span class="Apple-converted-space"></span> 9m9s</span></pre>

<pre class="p1"><span class="s1">**$** </span><span class="s2">kubectl get pods</span>

<span class="s2">NAME<span class="Apple-converted-space"></span> READY <span class="Apple-converted-space"></span> STATUS<span class="Apple-converted-space"></span> RESTARTS <span class="Apple-converted-space"></span> AGE</span>

<span class="s2">hazelcast-metrics-enabled-0 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 19m</span>

<span class="s2">hazelcast-metrics-enabled-1 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 18m</span>

<span class="s2">hazelcast-metrics-enabled-2 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 18m</span>

<span class="s2">hazelcast-metrics-enabled-3 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 2m55s</span>

<span class="s2">hazelcast-metrics-enabled-4 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 2m9s</span>

<span class="s2">hazelcast-metrics-enabled-5 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 83s</span>

<span class="s2">hazelcast-metrics-enabled-6 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 47s</span>

<span class="s2">hazelcast-metrics-enabled-7 <span class="Apple-converted-space"></span> 0/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 11s</span>

<span class="s2">hazelcast-metrics-enabled-mancenter-0 <span class="Apple-converted-space"></span> 1/1 <span class="Apple-converted-space"></span> Running <span class="Apple-converted-space"></span> 0<span class="Apple-converted-space"></span> 19m</span></pre>

# Conclusion

Autoscaling is an important feature for enterprises to save money and to cope with unexpected traffic coming to your deployments. However, configuring autoscaling needs to be done carefully because you might end up unnecessary scale up/down operations which might cost some instability in your system. In this blog post, we explained how you can use HPA with your Hazelcast Cluster based on Resource Metrics and Custom Metrics. If Kubernetes Pod/Node Level Cpu/Memory usage is fine for you then use Resource Metrics. If you have more specific requirements and you need to have hazelcast specific autoscaling capabilities, Custom Metrics is the answer.

# Software Versions

This is the list of software versions used in this blog post.

<pre class="p1"><span class="s1">**$** </span><span class="s2">helm ls</span>

<span class="s2">NAME <span class="Apple-converted-space"></span> REVISION UPDATED <span class="Apple-converted-space"></span> STATUS<span class="Apple-converted-space"></span> CHART <span class="Apple-converted-space"></span> APP VERSION NAMESPACE</span> 

<span class="s2">hazelcast-metrics-enabled 1 <span class="Apple-converted-space"></span> Mon Oct 21 15:25:06 2019 DEPLOYED hazelcast-1.9.2 <span class="Apple-converted-space"></span> 3.12.2 <span class="Apple-converted-space"></span> default<span class="Apple-converted-space">   </span></span>

<span class="s2">prometheus <span class="Apple-converted-space"></span> 1 <span class="Apple-converted-space"></span> Mon Oct 21 15:21:54 2019 DEPLOYED prometheus-9.1.1<span class="Apple-converted-space"></span> 2.11.1 <span class="Apple-converted-space"></span> monitoring</span>

<span class="s2">prometheus-adapter <span class="Apple-converted-space"></span> 1 <span class="Apple-converted-space"></span> Mon Oct 21 15:24:03 2019 DEPLOYED prometheus-adapter-1.3.0 v0.5.0 <span class="Apple-converted-space"></span> monitoring</span></pre>
