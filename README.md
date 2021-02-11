# helm-starter-istio

An Istio starter template for Helm.

Stop fiddling with Istio and Kubernetes YAML and start building. This starter sets up everything you need to get a container
running in Istio correctly the first time.

## Features

* Fastest way to get a new service into the Istio mesh
* Simplified Istio ingress configuration
* Simplified Istio port configuration
* ConfigMap driven by `values.yaml`, to facilitate easy Helm value overriding
* Creates the following Kubernetes and Istio objects
  * Service
  * Deployment
  * ConfigMap
  * VirtualService
  * DestinationRule
  
## Installation

* Clone into `~/.helm/starters` or,
* Install with the [`helm-starter`](https://github.com/salesforce/helm-starter) plugin.
  * `helm plugin install https://github.com/salesforce/helm-starter.git`
  * `helm starter fetch https://github.com/salesforce/helm-starter-istio.git`

## Usage

Pick the starter you want to use:

* `mesh-service` - creates a Helm chart for a mesh internal service (no ingress).
* `ingress-service` - creates a Helm chart for sevice exposed through an Istio ingress gateway.
* `mesh-egress` - creates a Helm chart for configuring mesh egress policies for external systems.
* `auth-policy` - creates a Helm chart for managing authorization policy within the mesh.

```sh
# Create a helm chart from the starter
> helm create NAME --starter helm-starter-istio/[starter-name]

# Deploy the helm chart to kubernetes
> helm template NAME | kubectl -apply -f -
```
