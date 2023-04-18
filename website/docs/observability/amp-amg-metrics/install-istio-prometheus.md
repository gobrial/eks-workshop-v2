---
title: "Integrate Istio with Prometheus"
sidebar_position: 20
---

In our setup Istio and Kiali are expecting Prometheus to be installed and available int the local cluster. For that we're going to use [Prometheus Stack]() Helm chart which is also available as a Blueprints Add-on. Since we're using Terraform, let's grab our Prometheus remote_write URL (it's workspace_url variable in the file) and add a file with the content below:

```
locals {
  name             = try(var.helm_config.name, "prometheus")
  namespace_name   = try(var.helm_config.namespace, "prometheus")
  create_namespace = try(var.helm_config.create_namespace, true) && local.namespace_name != "kube-system"

  # `namespace_name` is just the string representation of the namespace name
  # `namespace` is the name of the resultant namespace to use - created or not
  namespace = local.create_namespace ? kubernetes_namespace_v1.prometheus[0].metadata[0].name : local.namespace_name

  workspace_url          = INSERT_YOUR_PROMETHEUS_WS_URL
  ingest_service_account = "amp-ingest"
  ingest_iam_role_arn    = var.enable_amazon_prometheus ? module.irsa_amp_ingest[0].irsa_iam_role_arn : ""

  amp_gitops_config = var.enable_amazon_prometheus ? {
    roleArn            = local.ingest_iam_role_arn
    ampWorkspaceUrl    = local.workspace_url
    serviceAccountName = local.ingest_service_account
  } : {}
}

module "helm_addon" {
  source = "../helm-addon"

  manage_via_gitops = var.manage_via_gitops

  # https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus/Chart.yaml
  helm_config = merge(
    {
      name        = local.name
      chart       = local.name
      version     = "15.17.0"
      repository  = "https://prometheus-community.github.io/helm-charts"
      namespace   = local.namespace_name
      description = "Prometheus helm Chart deployment configuration"
      values = [templatefile("${path.module}/values.yaml", {
        operating_system = try(var.helm_config.operating_system, "linux")
      })]
    },
    var.helm_config
  )

  set_values = var.enable_amazon_prometheus ? [
    {
      name  = "serviceAccounts.server.name"
      value = local.ingest_service_account
    },
    {
      name  = "serviceAccounts.server.create"
      value = false
    },
    {
      name  = "serviceAccounts.server.annotations.eks\\.amazonaws\\.com/role-arn"
      value = local.ingest_iam_role_arn
    },
    {
      name  = "server.remoteWrite[0].url"
      value = local.workspace_url
    },
    {
      name  = "server.remoteWrite[0].sigv4.region"
      value = var.addon_context.aws_region_name
    }
  ] : []

  addon_context = var.addon_context
}

resource "kubernetes_namespace_v1" "prometheus" {
  count = local.create_namespace ? 1 : 0

  metadata {
    name = local.namespace_name
  }
}

# ------------------------------------------
# AMP Ingest Permissions
# ------------------------------------------
resource "aws_iam_policy" "ingest" {
  count = var.enable_amazon_prometheus ? 1 : 0

  name        = format("%s-%s", "amp-ingest", var.addon_context.eks_cluster_id)
  description = "Set up the permission policy that grants ingest (remote write) permissions for AMP workspace"
  policy      = data.aws_iam_policy_document.ingest.json
  tags        = var.addon_context.tags
}

module "irsa_amp_ingest" {
  source = "../../../modules/irsa"

  count = var.enable_amazon_prometheus ? 1 : 0

  create_kubernetes_namespace = false
  kubernetes_namespace        = local.namespace

  kubernetes_service_account    = local.ingest_service_account
  irsa_iam_policies             = [try(aws_iam_policy.ingest[0].arn, "")]
  irsa_iam_role_path            = var.addon_context.irsa_iam_role_path
  irsa_iam_permissions_boundary = var.addon_context.irsa_iam_permissions_boundary
  eks_cluster_id                = var.addon_context.eks_cluster_id
  eks_oidc_provider_arn         = var.addon_context.eks_oidc_provider_arn
}

# ------------------------------------------
# AMP Query Permissions
# ------------------------------------------
resource "aws_iam_policy" "query" {
  count = var.enable_amazon_prometheus ? 1 : 0

  name        = format("%s-%s", "amp-query", var.addon_context.eks_cluster_id)
  description = "Set up the permission policy that grants query permissions for AMP workspace"
  policy      = data.aws_iam_policy_document.query.json
  tags        = var.addon_context.tags
}

module "irsa_amp_query" {
  source = "../../../modules/irsa"

  count = var.enable_amazon_prometheus ? 1 : 0

  create_kubernetes_namespace = false
  kubernetes_namespace        = local.namespace

  kubernetes_service_account    = "amp-query"
  irsa_iam_policies             = [aws_iam_policy.query[0].arn]
  irsa_iam_role_path            = var.addon_context.irsa_iam_role_path
  irsa_iam_permissions_boundary = var.addon_context.irsa_iam_permissions_boundary
  eks_cluster_id                = var.addon_context.eks_cluster_id
  eks_oidc_provider_arn         = var.addon_context.eks_oidc_provider_arn
}
```

This file utilizes two existing Blueprints modules: Add-on and IRSA. We also create the required policy attachments and the namespace. Apply this one to the existing cluster and it'll be it: local in-cluster Prometheus will be sending the metrics to AMP.

Now, when we have metrics shipped to the centralized storage how do we distinguish between the clusters? For example, when we visualize `http_requests_total` we'll need to be able to separate the same metrics coming from cluster Blue and cluster Green. To address the issue we'll be using a universal label `cluster_name`. Before the local Prometheus ships metrics to AMP it'll be adding our custom metric with cluster name and we'll be able to filter the metrics by that label where needed.

Here's the example of how it works on Prometheus level:

```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: prometheus-server
  namespace: prometheus
spec:
  externalLabels:
    cluster_name: <cluster_name>
```