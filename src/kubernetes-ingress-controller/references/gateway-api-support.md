---
title: Gateway API Support
content_type: reference
alpha: false # This is the default, but is here for completeness

overrides:
  alpha:
    true:
      gte: 2.4.x
      lte: 2.5.x

---

The {{site.kic_product_name}} supports the following resources and features in the
[Gateway API](https://gateway-api.sigs.k8s.io/). By default:

- Core features are supported. If a core feature is not supported in the
  current version, it will be listed in `Unsupported Core Features`.
- Extended features are not supported. If an extended feature is supported in 
  current version, it will be listed in `Supported Extended Features`.

## Gateways and GatewayClasses

### Supported Versions

{% if_version gte: 2.6.x %}
- `v1beta1`
{% endif_version %}
{% if_version lte: 2.5.x %}
- `v1alpha2`
{% endif_version %}

## HTTP Routes

{{site.kic_product_name}}'s implementation of `HTTPRoute` supports multiple `BackendRefs` with a
round-robin load-balancing strategy applied by default across the
`Endpoints` or the `Services`. `BackendRefs` weights are now supported
to allow you to fine-tune the load-balancing between those backend services.

### Supported Versions

{% if_version gte: 2.6.x %}
- `v1beta1`
{% endif_version %}
{% if_version lte: 2.5.x %}
- `v1alpha2`
{% endif_version %}

### Supported Extended Features
- Supports `method` in route matches.

### Unsupported Core Features
- Does not support `queryParam` in route matches.
- Does not support `requestRedirect` in filters.

## TCP Routes

The {{site.kic_product_name}}'s implementation of `TCPRoute` supports multiple `BackendRefs` in
`TCPRoute` resources for load balancing.

### Supported Versions
- `v1alpha2`

## UDP Routes

The {{site.kic_product_name}}'s implementation of `UDPRoute` supports multiple `BackendRefs` in
`UDPRoute` resources for load balancing.

### Supported Versions
- `v1alpha2`

## TLS Routes

### Supported Versions
- `v1alpha2`

{% if_version gte:2.6.x %}
## Reference Grants

Kong implementation supports `ReferenceGrant` to allow routes to
reference backends in other namespaces in `BackendRefs`.

### Supported Versions
- `v1alpha2`
{% endif_version %}

{% if_version gte:2.4.x lte:2.5.x %}
## Reference Policies

The {{site.kic_product_name}}'s implementation supports using `ReferencePolicy` to allow routes to
reference backends in other namespaces in `BackendRefs`.

### Supported Versions
- `v1alpha2`
{% endif_version %}

{% if_version lte: 2.5.x %}
## Alpha limitations
{% endif_version %}
{% if_version gte: 2.6.x %}
## Beta limitations
{% endif_version %}

{{site.kic_product_name}} Gateway API support is a work in progress, and not all features of
Gateway APIs are supported. In particular:

{% if_version lte: 2.3.x %}
- `HTTPRoute` is the only supported route type. `TCPRoute`, `UDPRoute`, and `TLSRoute`
  are not yet implemented.
- `HTTPRoute` does not yet support multiple `backendRefs`. You cannot distribute
  requests across multiple Services.
{% endif_version %}
- `queryParam` matches are not supported.
{% if_version gte: 2.4.x %}
{% if_version lte: 2.5.x %}
- Gateway Listener configuration does not support `TLSConfig`. You can't
  load certificates for HTTP Routes and TLS Routes via Gateway
  configuration, and must either accept the default Kong certificate or add
  certificates and SNI resources manually via the admin API in DB-backed mode.
{% endif_version %}
{% endif_version %}
{% if_version gte: 2.5.x %}
- Gateways [are not provisioned automatically](/kubernetes-ingress-controller/{{page.kong_version}}/concepts/gateway-api#gateway-management).
- Kong [only supports a single Gateway per GatewayClass](/kubernetes-ingress-controller/{{page.kong_version}}/concepts/gateway-api#listener-compatibility-and-handling-multiple-gateways).
{% endif_version %}
- HTTPRoutes cannot be bound to a specific port using a [ParentReference](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1beta1.ParentReference).
  Kong serves all HTTP routes on all HTTP listeners.
