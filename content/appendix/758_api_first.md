---
title: "There's an API for That!"
chapter: false
weight: 758
draft: false
---

## Where are the APIs?

Kubernetes environments are based on web services.
There are many aspects to this. 

- workloads running as web services *in* your Kubernetes clusters
- web services that each Kubernetes cluster is made *of*
- additional web services used *around* your cluster


kubectl exec deployment/jumpbox -- curl http://demo.dev


- The containerized microservices that make up the workloads running *in* a Kubernetes cluster are usually a combination of one or more of the following:
    - Web-based application programming interfaces (APIs) based on JavaScript Object Notation (JSON), the eXtensible Markup Language (XML), or similar.
    - Web-based app components based on HTML, CSS, JavaScript, etc.
    - Networked data layer access using other non-web TCP-based protocols.
    - Networked messaging using UDP-based protocols.

## API First 

https://12factor.net
we could call it “There’s an API for that!” :) This book
[*Beyond the Twelve-Factor App*](https://www.oreilly.com/library/view/beyond-the-twelve-factor/9781492042631/) by Kevin Hoffman argues that API-First should have been one of the 12(+) factors.
- Security
- Telemetry - OpenTelemetry - logs, metrics, traces
- API First


Not only are the workloads *in* each Kubernetes cluster mostly web-based, but the operations and infrastructure *of* the Kubernetes cluster are based on web APIs.
This includes the Kubernetes administrative and management interfaces, including but not limited to:
- Kubernetes API into the cluster's control plane
- communications within the Kubernetes control plane
- Kubelet API from control plane to worker nodes
- Container Runtime Interface (CRI) from Kubelet to container runtime
Though much simpler than Kubernetes, even Docker operations are based on web APIs.
- The Docker API is similar to the Kubernetes CRI
In addition, most of the devops automation, monitoring, observability, proxy, and service mesh communications are based on web APIs. For example:
- Container readiness and liveness probes 
- Prometheus metrics scraping, collection, and aggregation
- Fluent Bit logging and forwarding

```
}$ kubectl get --raw / | head -10
{
  "paths": [
    "/.well-known/openid-configuration",
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
$ kubectl get --raw / | grep v1$
$ kubectl get --raw / | grep 'v1$'
$ kubectl get --raw / | grep '/v1",$'
    "/api/v1",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io/v1",
    "/apis/authorization.k8s.io/v1",
    "/apis/autoscaling/v1",
    "/apis/batch/v1",
    "/apis/certificates.k8s.io/v1",
    "/apis/coordination.k8s.io/v1",
    "/apis/discovery.k8s.io/v1",
    "/apis/events.k8s.io/v1",
    "/apis/networking.k8s.io/v1",
    "/apis/node.k8s.io/v1",
    "/apis/policy/v1",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1",
    "/apis/storage.k8s.io/v1",
```

```
$ kubectl get --raw /metrics | head -5
# HELP aggregator_openapi_v2_regeneration_count [ALPHA] Counter of OpenAPI v2 spec regeneration count broken down by causing APIService name and reason.
# TYPE aggregator_openapi_v2_regeneration_count counter
aggregator_openapi_v2_regeneration_count{apiservice="*",reason="startup"} 0
# HELP aggregator_openapi_v2_regeneration_duration [ALPHA] Gauge of OpenAPI v2 spec regeneration duration in seconds.
# TYPE aggregator_openapi_v2_regeneration_duration gauge
$ kubectl get --raw /metrics | head -10
# HELP aggregator_openapi_v2_regeneration_count [ALPHA] Counter of OpenAPI v2 spec regeneration count broken down by causing APIService name and reason.
# TYPE aggregator_openapi_v2_regeneration_count counter
aggregator_openapi_v2_regeneration_count{apiservice="*",reason="startup"} 0
# HELP aggregator_openapi_v2_regeneration_duration [ALPHA] Gauge of OpenAPI v2 spec regeneration duration in seconds.
# TYPE aggregator_openapi_v2_regeneration_duration gauge
aggregator_openapi_v2_regeneration_duration{reason="startup"} 0.610003644
# HELP aggregator_unavailable_apiservice [ALPHA] Gauge of APIServices which are marked as unavailable broken down by APIService name.
# TYPE aggregator_unavailable_apiservice gauge
aggregator_unavailable_apiservice{name="v1."} 0
aggregator_unavailable_apiservice{name="v1.admissionregistration.k8s.io"} 0
```

In summary, the workloads you run, the Kubernetes infrastructure itself, and your ability to monitor and measure any and all of that is based on web APIs. Therefore, you need to understand web APIs to work with Kubernetes beyond a conceptual level.

TODO: put together a good picture...

{{< mermaid >}}
graph 
kubectl([kubectl])
client0([Internet client]) 
client1([Mobile client]) 
subgraph cluster[Kubernetes cluster]
  pod0[Pod A]
  pod2[Pod B]
  pod3[Pod C]
end 
kubectl --> pod0
client0 --> pod2
client1 --> pod2
pod3 -->|internal client| pod2
classDef green fill:#9f6,stroke:#333,stroke-width:4px;
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue fill:#69f,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef cyan fill:#0ff,stroke:#333,stroke-width:4px;
classDef lavender fill:#fcf,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class kubectl,pod0 blue2;
class client0,client1,pod3 green;
class pod2 orange;
{{< /mermaid >}}

{{< mermaid >}}
graph LR
cloud9[Cloud 9<br>dev instance<br>in VPC]
subgraph pod[demo Pod]
  subgraph container[demo Container]
    apache((PID 1<br>apache2<br>root))
    demo((PID 23<br>php<br>www-data))
    kill((PID 35<br>kill 1<br>root))
  end
end
cloud9 -->|kubectl exec| kill
kill -->|kill 1| apache
apache -.-|child| demo
classDef orange fill:#f96,stroke:#333,stroke-width:4px;
classDef blue2 fill:#0af,stroke:#333,stroke-width:4px;
classDef yellow fill:#ff3,stroke:#333,stroke-width:2px;
classDef yelloworange fill:#fd3,stroke:#333,stroke-width:2px;
class cloud9 orange;
class pod yellow;
class container yelloworange;
class apache,demo,kill blue2;
{{< /mermaid >}}

