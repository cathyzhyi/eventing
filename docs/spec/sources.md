# Sources

A **Source** is any Kubernetes object that generates or imports an event and
relays that event to another endpoint on the cluster via
[CloudEvents](https://cloudevents.io). Sourcing events is critical to developing
a distributed system that reacts to events.

A **Sink** is an [_addressable_](./interfaces.md#addressable) resource that
takes responsibility for the event. A **Sink** could be a consumer of events, or
middleware. A **Sink** will respond with 2xx when it has accepted and processed
the event.

A Source:

- Represents an off- or on-cluster system, service or application that produces
  events to be consumed by a **Sink**.
- Produces or imports CloudEvents.
- Sends CloudEvents to the configured **Sink**.

In practice, sources are an abstract concept that allow us to create declarative
configurations through the usage of Custom Resource Definitions (CRDs) extending
Kubernetes. Source systems are configured by creating an instance of the
resource described by the CRD. It is up to the implementation of the source
author to understand the best way to realize the source application. This could
be as 1:1 deployments inside of Kubernetes per resource, as a single
multi-tenant application, or even an off-cluster implementation; or all
combinations in-between.

For operators of a Kubernetes cluster, there are two states to sources:

1. A Source CRD and controller (if required) have been installed into the
   cluster.

   A cluster with a source CRD installed allows the developers in the cluster to
   find and discover what is possible to source events from. This topic is
   expanded upon in the [Source CRDs](#source-crds) section.

1. A Source custom object has been created in the cluster.

   Once a developer creates an instance of a source and provides the necessary
   parameters, the source controller will realize this into whatever is required
   for that source for the current situation. While this resource is running, a
   cluster operator would like to inspect the resource without needing to be
   fully aware of the implementation. This is done by conforming to the
   [Source]() ducktype. This topic is expanded upon in the
   [Source Custom Objects](#source-custom-objects) section.

The goal of requiring CRD labels and running resource shapes is to enable
discovery and understanding of potential sources that could be leveraged in the
cluster. This structure also aids in understanding sources that exist and are
running in the cluster.

## Source CRDs

Sources are more useful if they are discoverable. Knative Sources MUST use a
standardized label to allow controllers and operators the ability to find which
CRDs are considered to be adhering to the
[Source](https://godoc.org/github.com/knative/pkg/apis/duck/v1#Source) ducktype.

CRDs that are to be understood as a `source` MUST be labeled:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  labels:
    duck.knative.dev/source: "true" # <-- required to be a source.
```

Labeling sources in this way allows for developers to filter the list of CRDs:

```shell
kubectl get crds -l duck.knative.dev/source=true
```

CRDs SHOULD be added to the `sources` category:

```yaml
spec:
  names:
    categories:
      - sources
```

By adding to the sources category, we give an easy way to list running sources
in the cluster with:

```shell
kubectl get sources
```

Source CRDs SHOULD provide additional printer columns to provide useful feedback
to cluster operators. For example, if the resource is long-lived, it would be a
good choice to show the `Ready` status and `Reason`, as well as the `Age` of the
resource.

```yaml
additionalPrinterColumns:
  - name: Ready
    type: string
    JSONPath: '.status.conditions[?(@.type=="Ready")].status'
  - name: Reason
    type: string
    JSONPath: '.status.conditions[?(@.type=="Ready")].reason'
  - name: Age
    type: date
    JSONPath: .metadata.creationTimestamp
```

### Source Validation

Sources MUST implement conditions with a `Ready` condition for long lived
sources, and `Succeeded` for batch style sources.

Sources MUST propagate the `sinkUri` to their status to signal to the cluster
where their events are being sent.

Knative has standardized on the following minimum OpenAPI definition of `status`
for Source CRDs:

```yaml
validation:
  openAPIV3Schema:
    properties:
      status:
        type: object
        properties:
          conditions:
            type: array
            items:
              type: object
              properties:
                lastTransitionTime:
                  type: string
                message:
                  type: string
                reason:
                  type: string
                severity:
                  type: string
                status:
                  type: string
                type:
                  type: string
              required:
                - type
                - status
          sinkUri:
            type: string
```

Please see
[SourceStatus](https://godoc.org/github.com/knative/pkg/apis/duck/v1#SourceStatus)
and [Condition](https://godoc.org/github.com/knative/pkg/apis/#Condition) for
more details.

Sources SHOULD provide OpenAPI validation for the `spec` field. At minimum
sources SHOULD have the following,

```yaml
validation:
  openAPIV3Schema:
    properties:
      spec:
        type: object
        required:
          - sink
        properties:
          sink:
            type: object
            description:
              "Reference to an object that will resolve to a domain name to use
              as the sink."
          ceOverrides:
            type: object
            description:
              "Defines overrides to control modifications of the event sent to
              the sink."
            properties:
              extensions:
                type: object
                description:
                  "Extensions specify what attributes are added or overridden on
                  the outbound event. Each `Extensions` key-value pair are set
                  on the event as an extension attribute independently."
```

<!-- TOOD(#1550): Follow-up with a registry implementation. -->

## Source RBAC

Sources are expected to be extensions onto Kubernetes. To prevent cluster
operators from duplicating RBAC for all accounts that will interact with
sources, cluster operators should leverage aggregated RBAC roles to dynamically
update the rights of controllers that are using common service accounts provided
by Knative. This allows Eventing controllers to check the status of source type
resources without being aware of the exact source type or implementation at
compile time.

The
[`source-observer` ClusterRole](../../config/200-source-observer-clusterrole.yaml)
looks like:

```yaml
# Use this aggregated ClusterRole when you need read "Sources".
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: source-observer
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        duck.knative.dev/source: "true" # Matched by source-observer ClusterRole
rules: [] # Rules are automatically filled in by the controller manager.
```

And new sources MUST include a ClusterRole as part of installing themselves into
a cluster:

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: foos-source-observer
  labels:
    duck.knative.dev/source: "true"
rules:
  - apiGroups:
      - example.com
    resources:
      - foos
    verbs:
      - get
      - list
      - watch
```

## Source Custom Objects

All Source custom objects MUST implement the
[Source](https://godoc.org/github.com/knative/pkg/apis/duck/v1#Source) ducktype.
Additional data in spec and status is explicitly permitted.

### duck.Spec

The `spec` field is expected to have the following minimum shape:

```go
type SourceSpec struct {
    // Sink is a reference to an object that will resolve to a domain name or a
    // URI directly to use as the sink.
    Sink apisv1alpha1.Destination `json:"sink,omitempty"`

    // CloudEventOverrides defines overrides to control the output format and
    // modifications of the event sent to the sink.
    // +optional
    CloudEventOverrides *CloudEventOverrides `json:"ceOverrides,omitempty"`
}
```

For a golang structure definition of `Sink` and `CloudEventsOverrides`, please
see [Destination](https://godoc.org/knative.dev/pkg/apis/v1alpha1#Destination),
and
[CloudEventOverrides](https://godoc.org/github.com/knative/pkg/apis/duck/v1#CloudEventOverrides).

### duck.Status

The `status` field is expected to have the following minimum shape:

```go
type SourceStatus struct {
    // inherits duck/v1 Status, which currently provides:
    // * ObservedGeneration - the 'Generation' of the Service that was last
    //   processed by the controller.
    // * Conditions - the latest available observations of a resource's current
    //   state.
    Status `json:",inline"`

    // SinkURI is the current active sink URI that has been configured for the
    // Source.
    // +optional
    SinkURI *apis.URL `json:"sinkUri,omitempty"`
}
```

For a full definition of `Status` and `SinkURI`, please see
[Status](https://godoc.org/github.com/knative/pkg/apis/duck/v1#Status), and
[URL](https://godoc.org/knative.dev/pkg/apis#URL).