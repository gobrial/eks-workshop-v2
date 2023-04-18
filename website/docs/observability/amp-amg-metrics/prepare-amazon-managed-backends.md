---
title: "Prepare and validate AMP backend"
sidebar_position: 10
---

# TODO: validate this agains EKS Add-on

An Amazon Managed Service for Prometheus workspace is already created for you. You should be able to see it in the console:

https://console.aws.amazon.com/prometheus/home#/workspaces

To view the workspace, click on the **All Workspaces** tab on the left control panel. Select the workspace that starts with **eks-workshop** and you can view several tabs under the workspace such as rules management, alert manager etc.

Let's verify the successful ingestion of the metrics using `awscurl`:

```bash
$ awscurl -X POST --region $AWS_DEFAULT_REGION --service aps "${AMP_ENDPOINT}api/v1/query?query=up" | jq '.data.result[1]'
{
  "metric": {
    "__name__": "up",
    "account_id": "1234567890",
    "beta_kubernetes_io_arch": "amd64",
    "beta_kubernetes_io_instance_type": "m5.large",
    "beta_kubernetes_io_os": "linux",
    "cluster": "eks-workshop",
    "eks_amazonaws_com_capacityType": "ON_DEMAND",
    "eks_amazonaws_com_nodegroup": "managed-ondemand-2022110404042617720000001b",
    "eks_amazonaws_com_nodegroup_image": "ami-01dfb5782bffd09d6",
    "eks_amazonaws_com_sourceLaunchTemplateId": "lt-0566ef61fb851d6e1",
    "eks_amazonaws_com_sourceLaunchTemplateVersion": "1",
    "failure_domain_beta_kubernetes_io_region": "us-west-2",
    "failure_domain_beta_kubernetes_io_zone": "us-west-2c",
    "instance": "ip-10-42-12-99.us-west-2.compute.internal",
    "job": "kubernetes-kubelet",
    "k8s_io_cloud_provider_aws": "ffc60533e6d069826fca0578b02694a2",
    "kubernetes_io_arch": "amd64",
    "kubernetes_io_hostname": "ip-10-42-12-99.us-west-2.compute.internal",
    "kubernetes_io_os": "linux",
    "node_kubernetes_io_instance_type": "m5.large",
    "region": "us-west-2",
    "topology_ebs_csi_aws_com_zone": "us-west-2c",
    "topology_kubernetes_io_region": "us-west-2",
    "topology_kubernetes_io_zone": "us-west-2c",
    "workshop_default": "yes"
  },
  "value": [
    1667597359,
    "1"
  ]
}
```

#TODO: autotest must be able to catch absence of `cluster_name`

An instance of Grafana has been pre-installed in your EKS cluster. To access it you first need to retrieve the URL:

```bash hook=check-grafana
$ kubectl get ingress -n grafana grafana -o=jsonpath='{.status.loadBalancer.ingress[0].hostname}'
k8s-grafana-grafana-123497e39be-2107151316.us-west-2.elb.amazonaws.com
```

Opening this URL in a browser will bring up a login screen. 

![Grafana dashboard](./assets/grafana-login.png)

To retrieve the credentials for the user query the secret created by the Grafana helm chart:

```bash
$ kubectl get -n grafana secrets/grafana -o=jsonpath='{.data.admin-user}' | base64 -d
$ kubectl get -n grafana secrets/grafana -o=jsonpath='{.data.admin-password}' | base64 -d
```

After logging into the Grafana console, let's take a look at the datasources section. You should see the Amazon Managed Service for Prometheus workspace configured as a datasource already.

![Amazon Managed Service for Prometheus Datasource](./assets/datasource.png)
