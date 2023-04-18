---
title: "Centralized metrics with Amazon Managed Prometheus and Amazon Managed Grafana"
sidebar_position: 70
sidebar_custom_props: {"module": true}
---

:::tip Before you start
Prepare your environment for this section:

```bash timeout=300 wait=30
$ reset-environment 
```

:::

Many organizations have more than a single EKS cluster nowadays. With Istio enabling operators to setup multi-cluster mesh this has even more value. But we don't want our metrics disappear when we replace clusters. To address this issue we'll use centralized metrics storage [Amazon Managed Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html) (AMP) along with [Amazon Managed Grafana](https://docs.aws.amazon.com/grafana/latest/userguide/getting-started-with-AMG.html) (AMG) to visualize the metrics.

In Istio, by default, all the systems expect the [Prometheus server and Grafana to be installed in the same cluster in the istio-system namespace](https://kiali.io/docs/configuration/p8s-jaeger-grafana/prometheus/). Kiali (Isio's UI Dashboard) then uses both systems to interact with the metrics and dashboards respectively. To preserve all the functionality we'll be keeping the default Prometheus/Grafana in place but also we'll add AMP/AMG to have the same set of metrics centralized so if the cluster is gone the metrics stays in the centralized storage. This implementation is called ['remote_write'](https://prometheus.io/docs/practices/remote_write/) in Prometheus and is available in all configurations including the one Istio has by default.

![Design Schema](./assets/design_schema.jpg)
