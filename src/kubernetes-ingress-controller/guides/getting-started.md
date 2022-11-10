---
title: Getting started with the Kubernetes Ingress Controller
---

## Installation

Please follow the [deployment](/kubernetes-ingress-controller/{{page.kong_version}}/deployment/overview) documentation to install
the {{site.kic_product_name}} onto your Kubernetes cluster.

If you wish to use the Gateway APIs examples, you must first [install the
Gateway API resources](https://gateway-api.sigs.k8s.io/guides/#installing-gateway-api)
and restart your Deployment after:

{% navtabs codeblock %}
{% navtab Command %}
```bash
kubectl rollout restart -n NAMESPACE deployment DEPLOYMENT_NAME
```
{% endnavtab %}
{% navtab Response %}
```text
deployment.apps/DEPLOYMENT_NAME restarted
```
{% endnavtab %}
{% endnavtabs %}

## Testing connectivity to Kong

This guide assumes that `PROXY_IP` environment variable is
set to contain the IP address or URL pointing to Kong.
If you've not done so, please follow one of the
[deployment guides](/kubernetes-ingress-controller/{{page.kong_version}}/deployment/overview) to configure this environment variable.

If everything is setup correctly, making a request to Kong should return back
a HTTP `404 Not Found` status code.

{% navtabs codeblock %}
{% navtab Command %}
```bash
curl -i $PROXY_IP
```
{% endnavtab %}
{% navtab Response %}
```text
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 48
X-Kong-Response-Latency: 0
Server: kong/3.0.0

{"message":"no Route matched with those values"}
```
{% endnavtab %}
{% endnavtabs %}

This is expected since Kong doesn't know how to proxy the request yet.

## Deploy an upstream application

To proxy requests, you need an upstream application to proxy to. Deploying this
echo server provides a simple application that returns information about the
Pod it's running in:

{% navtabs codeblock %}
{% navtab Command %}
```bash
kubectl apply -f https://bit.ly/echo-service
```
{% endnavtab %}
{% navtab Response %}
```text
service/echo created
deployment.apps/echo created
```
{% endnavtab %}
{% endnavtabs %}

## Add whatever we call this

Ingress and Gateway APIs controllers need configuration indicating which set of
routing configuration they should recognize, to allow multiple controller to
coexist in the same cluster. Before creating individual routes, you need to
create class configuration to associate routes with:

{% navtabs api %}
{% navtab Ingress %}
{% navtabs codeblock %}
{% navtab Command %}
```bash
echo "
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: kong
spec:
  controller: ingress-controllers.konghq.com/kong
" | kubectl apply -f -
```
{% endnavtab %}
{% navtab Response %}
```text
Warning: resource ingressclasses/kong is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
ingressclass.networking.k8s.io/kong configured
```
Official distributions of {{site.kic_product_name}} come with a `kong`
IngressClass by default, but without `apply` history, hence the warning above.
{% endnavtab %}
{% endnavtabs %}

{{site.kic_product_name}} recognizes the `kong` IngressClass by default.
Setting the `CONTROLLER_INGRESS_CLASS` environment variable to another value
overrides this.

{% endnavtab %}
{% navtab Gateway APIs %}
{% navtabs codeblock %}
{% navtab Command %}
{% if_version lte: 2.5.x %}
```bash
echo "
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: kong
spec:
  controllerName: konghq.com/kic-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: kong
  annotations:
    konghq.com/gateway-unmanaged: kong/kong-proxy
spec:
  gatewayClassName: kong
  listeners:
  - name: proxy
    port: 80
    protocol: HTTP
  - name: proxy-ssl
    port: 443
    protocol: HTTPS
" | kubectl apply -f -
```
{% endif_version %}
{% if_version gte: 2.6.x %}
```bash
echo "
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: 'true'
spec:
  controllerName: konghq.com/kic-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: kong
spec:
  gatewayClassName: kong
  listeners:
  - name: proxy
    port: 80
    protocol: HTTP
  - name: proxy-ssl
    port: 443
    protocol: HTTPS
" | kubectl apply -f -
```
{% endnavtab %}
{% navtab Response %}
```text
gatewayclass.gateway.networking.k8s.io/kong created
gateway.gateway.networking.k8s.io/kong created
```
{% endnavtab %}
{% endnavtabs %}

{{site.kic_product_name}} recognizes GatewayClasses with `controllerName:
konghq.com/kic-gateway-controller` by default. Setting the
`CONTROLLER_GATEWAY_API_CONTROLLER_NAME` environment variable to another value
overrides this.

{% endnavtab %}
{% endnavtabs %}

{{site.kic_product_name}} recognizes the `kong` IngressClass and
`konghq.com/kic-gateway-controller` GatewayClass
by default. Setting the `CONTROLLER_INGRESS_CLASS` environment variable to
another value overrides this.

## Add routing configuration

Create routing configuration to proxy `/echo` requests to the echo server:

{% navtabs api %}
{% navtab Ingress %}
{% navtabs codeblock %}
{% navtab Command %}
```bash
echo "
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  annotations:
    konghq.com/strip-path: 'true'
spec:
  ingressClassName: kong
  rules:
  - http:
      paths:
      - path: /echo
        pathType: ImplementationSpecific
        backend:
          service:
            name: echo
            port:
              number: 80
" | kubectl apply -f -
```
{% endnavtab %}
{% navtab Response %}
```text
ingress.networking.k8s.io/echo created
```
{% endnavtab %}
{% endnavtabs %}
{% endnavtab %}
{% navtab Gateway APIs %}
{% navtabs codeblock %}
{% navtab Command %}
```bash
echo "
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: echo
  annotations:
    konghq.com/strip-path: 'true'
spec:
  parentRefs:
  - name: kong
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /echo
    backendRefs:
    - name: echo
      kind: Service
      port: 80
" | kubectl apply -f -
```
{% endnavtab %}
{% navtab Response %}
```text
httproute.gateway.networking.k8s.io/echo created
```
{% endnavtab %}
{% endnavtabs %}
{% endnavtab %}
{% endnavtabs %}

Test the Ingress rule:

{% navtabs codeblock %}
{% navtab Command %}
```bash
curl -i $PROXY_IP/echo
```
{% endnavtab %}
{% navtab Response %}
```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Date: Thu, 10 Nov 2022 22:10:40 GMT
Server: echoserver
X-Kong-Upstream-Latency: 0
X-Kong-Proxy-Latency: 0
Via: kong/3.0.0



Hostname: echo-fc6fd95b5-6lqnc

Pod Information:
	node name:	kind-control-plane
	pod name:	echo-fc6fd95b5-6lqnc
	pod namespace:	default
	pod IP:	10.244.0.9
...
```
{% endnavtab %}
{% endnavtabs %}

If everything is deployed correctly, you should see the above response.
This verifies that Kong can correctly route traffic to an application running
inside Kubernetes.

## Using plugins in Kong

Setup a KongPlugin resource:

{% navtabs codeblock %}
{% navtab Command %}
```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-id
config:
  header_name: my-request-id
  echo_downstream: true
plugin: correlation-id
" | kubectl apply -f -
```
{% endnavtab %}
{% navtab Response %}
```text
kongplugin.configuration.konghq.com/request-id created
```
{% endnavtab %}
{% endnavtabs %}

Update your route configuration to use the new plugin:

{% navtabs api %}
{% navtab Ingress %}
{% navtabs codeblock %}
{% navtab Command %}
```bash
kubectl annotate ingress echo konghq.com/plugins=request-id
```
{% endnavtab %}
{% navtab Response %}
```text
ingress.networking.k8s.io/echo annotated
```
{% endnavtab %}
{% endnavtabs %}
{% endnavtab %}
{% navtab Gateway APIs %}
{% navtabs codeblock %}
{% navtab Command %}
```bash
kubectl annotate httproute echo konghq.com/plugins=request-id
```
{% endnavtab %}
{% navtab Response %}
```text
httproute.gateway.networking.k8s.io/echo annotated
```
{% endnavtab %}
{% endnavtabs %}
{% endnavtab %}
{% endnavtabs %}

Kong will now apply your plugin configuration to all routes associated with
this resource. To test, it send another request through the proxy:

{% navtabs codeblock %}
{% navtab Command %}
```bash
curl -i $PROXY_IP/echo
```
{% endnavtab %}
{% navtab Response %}
```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
Date: Thu, 10 Nov 2022 22:33:14 GMT
Server: echoserver
my-request-id: ea87894d-7f97-4710-84ae-cbc608bb8107#2
X-Kong-Upstream-Latency: 0
X-Kong-Proxy-Latency: 0
Via: kong/3.0.0

Hostname: echo-fc6fd95b5-6lqnc

Pod Information:
	node name:	kind-control-plane
	pod name:	echo-fc6fd95b5-6lqnc
	pod namespace:	default
	pod IP:	10.244.0.9
...
Request Headers:
    ...
	my-request-id=ea87894d-7f97-4710-84ae-cbc608bb8107#2
...
```
{% endnavtab %}
{% endnavtabs %}

Requests that match the `echo` Ingress or HTTPRoute now include a
`my-request-id` header with a unique ID in both their request headers upstream
and their response headers downstream.

## Using plugins on Services

Kong can also apply plugins to Services. This allows you execute the same
plugin configuration on all requests to that Service, without configuring the
same plugin on multiple Ingresses.

Create a KongPlugin resource:

{% navtabs codeblock %}
{% navtab Command %}
```bash
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rl-by-ip
config:
  minute: 5
  limit_by: ip
  policy: local
plugin: rate-limiting
" | kubectl apply -f -
```
{% endnavtab %}
{% navtab Response %}
```text
kongplugin.configuration.konghq.com/rl-by-ip created
```
{% endnavtab %}
{% endnavtabs %}

Next, apply the `konghq.com/plugins` annotation to the Kubernetes Service
that needs rate-limiting:

{% navtabs codeblock %}
{% navtab Command %}
```bash
kubectl annotate service echo konghq.com/plugins=rl-by-ip
```
{% endnavtab %}
{% navtab Response %}
```text
service/echo annotated
```
{% endnavtab %}
{% endnavtabs %}

Kong will now enforce a rate limit to requests proxied to this Service:

{% navtabs codeblock %}
{% navtab Command %}
```bash
curl -i $PROXY_IP/echo
```
{% endnavtab %}
{% navtab Response %}
```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=UTF-8
Transfer-Encoding: chunked
Connection: keep-alive
X-RateLimit-Remaining-Minute: 4
RateLimit-Limit: 5
RateLimit-Remaining: 4
RateLimit-Reset: 13
X-RateLimit-Limit-Minute: 5
Date: Thu, 10 Nov 2022 22:47:47 GMT
Server: echoserver
my-request-id: ea87894d-7f97-4710-84ae-cbc608bb8107#3
X-Kong-Upstream-Latency: 1
X-Kong-Proxy-Latency: 1
Via: kong/3.0.0



Hostname: echo-fc6fd95b5-6lqnc

Pod Information:
	node name:	kind-control-plane
	pod name:	echo-fc6fd95b5-6lqnc
	pod namespace:	default
	pod IP:	10.244.0.9
...
```
{% endnavtab %}
{% endnavtabs %}

## Next steps

* To learn how to secure proxied routes, see the [ACL and JWT Plugins Guide](/kubernetes-ingress-controller/{{page.kong_version}}/guides/configure-acl-plugin/).
* The [External Services Guide](/kubernetes-ingress-controller/{{page.kong_version}}/guides/using-external-service/) explains how to proxy services outside of your Kubernetes cluster.
{% if_version gte:2.4.x %}
* [Gateway API](https://gateway-api.sigs.k8s.io/) is a set of resources for
configuring networking in Kubernetes. The Kubernetes Ingress Controller supports Gateway API by default. To learn how to use Gateway API supported by the Kubernetes Ingress Controller, see [Using Gateway API](/kubernetes-ingress-controller/{{page.kong_version}}/guides/using-gateway-api).
{% endif_version %}
