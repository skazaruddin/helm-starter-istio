# Ingress Service Helm Chart

This Helm chart installs a service in both Kubernetes and Istio, and exposes
it outside the cluster via an Istio ingress gateway.

## Installation

```sh
> helm template --namespace=[namespace] [chartname] | kubectl apply -f -
```

## Values.yaml

All configuration for this installation is managed in `values.yaml`. Configuration
values can be overriden individually at installation using Helm's `--set` command
line option.

### Service Identity

These three values control the names of generated Kubernetes and Istio objects,
and are used to ensure commont Kubernetes labeling. These values are used to populate
labels that allow for selecting all components of a particular system or service.

* `system`, `service`, `version` - These values describe _what_ this service and
  what it should be named. For example: `my-website`, `web-server`, `2`.

### Container Values

These settings control from where and how your service's docker image is acquired.

* `image.repository` - The docker repo and image to pull.
* `image.tag` - The docker image tag to pull.
* `image.imagePullPolicy` - Kubernetes image pull policy.

### Service Account Values

Istio request authorization requires that each service have a unique service account
identity to functuion correctly.

* `serviceAccount.name` - The Kubernetes service account your service will run under.
* `serviceAccount.create` - Optionally, this chart can generate the service account.
  If false, the service's service account must be pre-existing.

### Replica Values

These settings control service replicas, disruption budgets, and autoscaling.

* `replicaCount` - The initial number of replicas to start after installing this
  chart.
* `maxUnavailable` - The maximum number of intentionally unavailable pods as
  controlled by a `PodDisruptionBudget`.
* `autoscaling.minReplicas` - The minimum number of replicas to run under the
  control of a `HorizontalPodAutoscaler`.
* `autoscaling.maxReplicas` - The maximum number of replicas to run under the
  control of a `HorizontalPodAutoscaler`.
* `autoscaling.targetAverageCpuUtilization` - The CPU utilization target
  used by the `HorizontalPodAutoscaler` to make autoscale decisions.

### Port Values

These settings expose network ports through Kubernetes and Istio. Ports are
listed as an array.

* `ports.name` - The unique name for this port. Port names _should_ comply
  with Istio port naming standards by including their protocol in the name.
  Protocol detection works for HTTP and HTTP/2, but other protocols need help.
  <https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/>
* `ports.port` - The port number presented _outside_ your container. This is the
  port Istio and Kubernetes will use when referring to your service's port.
* `ports.targetPort` - The internal port number _inside_ your container. Kubernetes
  will map the outside port to the inside port when routing traffic to your container.

### Ingress Values

These settings configure how Istio exposes your service throug an Istio ingress
gateway. They assume the Istio ingress gateway is installed, and an Istio
`Gateway` object has been configured in the mesh.

* `ingressGateway.name` - The namespace/name of the Istio `Gateway` object through
  which this service should be exposed.
* `ingressGateway.host` - Bind this service's ingress configuration to a hostname.
* `ingressGateway.matchPrefix` - A string array of REST route prefixes this service
  matches. gRPC services are matched as `/protoNamespace/protoService/*`.

### Resiliency Values

These settings control Istio's resiliency configuration for your service. This
includes timeouts, circuit breakers, retires, and outlier detection.

* `overallTimeout` - The "top-level" timeout enforced when clients call your
  service. This timeout is inclusive of retries.
* `retries.*` - Istio configuration for client retry policy. See
  [Istio retry documentation](https://istio.io/latest/docs/reference/config/networking/virtual-service/#HTTPRetry) for values. Note: if retries are configured,
  an `overallTimeout` greater than the sum of all retries must be used.
* `outlierDetection.*` - Istio configuration for client circuit breaker configuration.
  See [Istio outlier detection documentation](https://istio.io/latest/docs/reference/config/networking/destination-rule/#OutlierDetection) for details.

### Kubernetes Pod Values

These settings configure your service's resource constraints and health check
probes. They ensure your service is a well behaved consumer of shared Kubernetes
resources.

* `resources.*` - Kubernetes resource request and limit configuration. See
  [Kubernetes resource documentation](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for values.
* `probes.*` - Kubernetes probe configuration. See [Kubernetes probe documentation](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) for values.

### ConfigMap Values

These optional settings are used to populate and mount a configmap for your
service. When the generated config map changes, the associated service is automatically
resterted using a rolling restart. Generating the configmap from Helm chart values
is useful because it allows you to modify config map values durring installation
using Helm `--set` directives.

* `configMap.mountPath` - The directory inside your pod to mount the config map.
* `configMap.fileName` - The file name of the config map, when mounted in the pod.
* `configMap.content.*` - YAML keys and values under `content` are copied verbatim
  into the configmap's content.

## IRSA on Kubernetes

Allow containers to assume IAM role via AWS STS Service, to connect to AWS provided services via IAM Role instead of credentials.
To allow a Kubernetes ServiceAccount to assume an IAM role, you need to configure IAM Roles for Service Accounts (IRSA) in AWS. 

{{- if .Values.serviceAccount.irsa.create }}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.serviceAccount.name | lower | quote }}
  labels:
    app.kubernetes.io/name: {{ .Values.serviceAccount.name | lower | quote }}
    app.kubernetes.io/part-of: {{ .Values.system | quote }}
    app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::YOUR_AWS_ACCOUNT_ID:role/YOUR_IAM_ROLE_ARN
{{- end }}


This involves creating an IAM role with the necessary policies and setting up a trust relationship between the IAM role 
and the Kubernetes ServiceAccount. Here’s how we can update your Helm template to support this.

Step 1: Create the IAM Role and Policy
First, create an IAM role with a trust relationship that allows your Kubernetes ServiceAccount to assume it. 
You can do this via the AWS Management Console, CLI, or a Terraform script. Here’s an example using the AWS CLI:

#### Create a policy document that allows the necessary actions
cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_AWS_ACCOUNT_ID:oidc-provider/YOUR_OIDC_PROVIDER"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "YOUR_OIDC_PROVIDER:sub": "system:serviceaccount:YOUR_KUBERNETES_NAMESPACE:YOUR_KUBERNETES_SERVICE_ACCOUNT_NAME"
        }
      }
    }
  ]
}
EOF

#### Create the IAM role
aws iam create-role --role-name YOUR_ROLE_NAME --assume-role-policy-document file://trust-policy.json

#### Attach the necessary policies to the role
aws iam attach-role-policy --role-name YOUR_ROLE_NAME --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

#### NOTE: Before creating the IAM Policy and triggering the command to create role, replace YOUR_AWS_ACCOUNT_ID, YOUR_OIDC_PROVIDER in AWS EKS Cluster, YOUR_KUBERNETES_NAMESPACE, YOUR_KUBERNETES_SERVICE_ACCOUNT_NAME, and YOUR_ROLE_NAME with your actual values.
