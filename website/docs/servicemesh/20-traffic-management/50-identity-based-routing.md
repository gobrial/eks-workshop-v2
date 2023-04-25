---
title: Identity Based Routing
sidebar_position: 40
weight: 5
---

In this task, you use Istio to send 100% of the traffic to `ui-v1`. You then set a rule to selectively send traffic to `ui-v2` based on the user-agent added to the end-user header request by the UI service. 

The User-Agent request header is a characteristic string that lets servers and network peers identify the application, operating system, vendor, and/or version of the requesting user agent.

In this case, all traffic from a user accessing the `ui` service from Safari will be routed to the deployment `ui-v2`, but all remaining user traffic from other browsers goes to `ui-v1`.

To check which version of the UI traffic is routed to, the grep banner at the top of the page should display the pod name which includes the deployment version.

Now, let's open the retail store sample using the browser:
```bash
$ echo $ISTIO_IG_HOSTNAME/home
```
Output Similar to:
```bash
http://af08b29901d054f489ffb6473b1593a9-510276527.us-east-1.elb.amazonaws.com/home
```

Currently, your virtualservice routes to all 3 versions of the ui.

The following virtualservice manifest file shows how you can create an identity-based route.

```
kubectl apply -k workspace/manifests/servicemesh/20-traffic-management/50-identity-based-routing
```

```
workspace/manifests/servicemesh/20-traffic-management/50-identity-based-routing/virtual-service-recommendation-v1.yaml
```
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ui-virtualservice
spec:
  hosts:
  - "*"
  gateways:
  - ui-gateway
  http:
  - match:
    - headers:
        user-agent:
          regex: .*Safari.*
    route:
    - destination:
        host: ui
        subset: v2
  - route:
    - destination:
        host: ui
        subset: v1
---
```

Navigate to the following URL from the browser:

```bash
$ echo $ISTIO_IG_HOSTNAME/home
```

Test with a Safari (or even Chrome on Mac since it includes Safari in the string). Safari only sees `ui-v2` responses from the `ui` service

And test with a Firefox browser, it should only see `ui-v1` responses from the `ui` service.

Once the browser is loaded, you will notice that the grep banner on the displayed page will now only route to `ui-v2`.

![ui-grep-banner](../assets/ui-grep-banner.png)

You have successfully configured Istio to route traffic based on user identity.


