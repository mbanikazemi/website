+++
date = "2018-07-31T12:00:00-07:00"
title = "Prometheus"
description = "Prometheus collects and stores all the Linkerd metrics. It is a component of the control plane and can be integrated with existing metric systems, such as an existing Prometheus install."
aliases = [
  "/2/prometheus/"
]
[menu.l5d2docs]
  name = "Prometheus"
  parent = "observability"
+++

If you have an existing Prometheus cluster, it is very easy to export Linkerd's
rich telemetry data to your cluster.  Simply add the following item to your
`scrape_configs` in your Prometheus config file (replace `{{.Namespace}}` with
the namespace where Linkerd is running):

```yaml
- job_name: 'linkerd'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
      names: ['{{.Namespace}}']

  relabel_configs:
  - source_labels:
    - __meta_kubernetes_pod_container_name
    action: keep
    regex: ^prometheus$

  honor_labels: true
  metrics_path: '/federate'

  params:
    'match[]':
      - '{job="linkerd-proxy"}'
      - '{job="linkerd-controller"}'
```

That's it!  Your Prometheus cluster is now configured to scrape Linkerd's
metrics.

Linkerd's proxy metrics will have the label `job="linkerd-proxy"`.  Linkerd's
control-plane metrics will have the label `job="linkerd-controller"`.

For more information on specific metric and label definitions, have a look at
[Proxy Metrics](../proxy-metrics).

For more information on Prometheus' `/federate` endpoint, have a look at the
[Prometheus federation docs](https://prometheus.io/docs/prometheus/latest/federation/).

# Exporting to other data stores

If you want to export to a data store other than Prometheus, you can query
Linkerd's Prometheus’ instance directly, via the Federation API or JSON API.

### Prometheus Federation API

```bash
curl -G \
  --data-urlencode 'match[]={job="linkerd-proxy"}' \
  --data-urlencode 'match[]={job="linkerd-controller"}' \
  http://prometheus.linkerd.svc.cluster.local:9090/federate
```

Note: if your data store is outside the Kubernetes cluster, it is likely that
you'll want to setup
[ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
at a domain name of your choice with authentication

### Prometheus JSON API

Similar to the `/federate` API, Prometheus provides a JSON API to retrieve all
metrics:

```bash
curl http://prometheus.linkerd.svc.cluster.local:9090/api/v1/query?query=request_total
```

# Querying the proxy

If you want to query a Linkerd proxy directly, you can use its `/metrics`
endpoint.

Each Linkerd proxy, injected as a sidecar with your application, provides
application-level metrics for all requests transiting through your application's
pod.

For example, to view `/metrics` from a single Linkerd proxy, running in the
`linkerd` namespace:

```bash
kubectl -n linkerd port-forward \
  $(kubectl -n linkerd get pods \
    -l linkerd.io/control-plane-ns=linkerd \
    -o jsonpath='{.items[0].metadata.name}') \
  4191:4191
```

and then:

```bash
curl localhost:4191/metrics
```
