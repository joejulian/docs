# Splitter

## Splitter overview

An event `Splitter` is part of the TriggerMesh routing solution. It has the simple purpose of splitting JSON arrays into multiple [CloudEvents](https://cloudevents.io/) for further processing.

!!! tip
    You can verify that the API is available with the following command:

    ```console
    $ kubectl get crd splitters.routing.triggermesh.io
    NAME                               CREATED AT
    splitters.routing.triggermesh.io   2021-10-06T09:01:38Z
    ```

    You can also explore the API specification with:
    ```console
    $ kubectl explain splitter
    ```

## Splitter tutorial on Kubernetes

![](../assets/images/splitter.png)

Let's create the required objects:

- [x] The `sockeye` target which serves as an event display.
- [x] The `PingSource` which produces a JSON array in its payload.
- [x] The `Splitter` to generate the multiple events.

### Event display

First of all, we need to have a tool to see the split events. Create a `sockeye`
service by saving the following YAML manifest in a file called `sockeye.yaml` and applying it to your Kubernetes cluster:

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: sockeye
spec:
  template:
    spec:
      containers:
        - image: docker.io/n3wscott/sockeye:v0.7.0@sha256:e603d8494eeacce966e57f8f508e4c4f6bebc71d095e3f5a0a1abaf42c5f0e48
```

```
kubectl apply -f sockeye.yaml
```

Open the web interface in a browser at the URL found with the following command:

```shell
$ kubectl get ksvc sockeye -o=jsonpath='{.status.url}'
```

### Events producer

Next we create the [PingSource](https://knative.dev/docs/developer/eventing/sources/ping-source) which produces CloudEvents that contain a list in their payload. Save the following YAML manifest in a file and apply it to your Kubernetes cluster with `kubectl apply`.

```yaml
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: ps-splitter-demo
spec:
  schedule: "*/1 * * * *"
  contentType: "application/json"
  data: '{
      "message":"hello",
      "items":[
          {
              "id":5,
              "name":"foo"
          },{
              "id":10,
              "name":"bar"
          }
        ]
      }'
  sink:
    ref:
      apiVersion: routing.triggermesh.io/v1alpha1
      kind: Splitter
      name: splitter-demo
```

### The Splitter

Finally, create the `Splitter` that will produce two events from every single CloudEvent received from the PingSource by saving the following YAML manifest and applying it to your Kubernetes cluster.

```yaml
apiVersion: routing.triggermesh.io/v1alpha1
kind: Splitter
metadata:
  name: splitter-demo
spec:
  path: items
  ceContext:
    type: foo.bar.type
    source: splitter
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: sockeye
```

Note that you define the path where you are going to find a list in the incoming event and you define the CloudEvent attributes of the generated events (i.e _splitter_ as the source and _foo.bar.type_ as the type).

Verify that your splitter is ready with `kubectl` like so:

```console
$ kubectl get splitter
NAME            ADDRESS                                                          READY   REASON
splitter-demo   http://splitter-adapter.sebgoa.svc.cluster.local/splitter-demo   True
```

In the sockeye application you will see two individual events that have been generated from the original list emitted by the source. The snapshot below shows you what you should see:

![](../assets/images/splitter-sockeye.png)

!!! tip "Play with your Splitter as Code"
    You can play around by modifying the `Splitter` object and re-applying it with `kubectl`. This gives you a declarative event splitter which you can manage with your [GitOps workflow](https://www.weave.works/technologies/gitops/)

## Another Splitter example for Kubernetes

```yaml
apiVersion: routing.triggermesh.io/v1alpha1
kind: Splitter
metadata:
  name: splitter-test
spec:
  path: items
  ceContext:
    type: foo.bar.type
    source: splitter
    extensions:
      key1: value1
      key2: value2
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
---
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  annotations:
    eventing.knative.dev/broker.class: MTChannelBasedBroker
  name: default
spec:
---
apiVersion: sources.knative.dev/v1beta2
kind: PingSource
metadata:
  name: ps-splitter-demo
spec:
  contentType: application/json
  data: '{"message":"hello","items":[{"id":5,"name":"foo"},{"id":10,"name":"bar"}]}'
  schedule: '*/1 * * * *'
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: to-splitter
spec:
  broker: default
  filter:
    attributes:
      type: dev.knative.sources.ping
  subscriber:
    ref:
      apiVersion: routing.triggermesh.io/v1alpha1
      kind: Splitter
      name: splitter-test
---
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: sockeye
spec:
  template:
    spec:
      containers:
      - image: docker.io/n3wscott/sockeye:v0.7.0@sha256:e603d8494eeacce966e57f8f508e4c4f6bebc71d095e3f5a0a1abaf42c5f0e48
---
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: to-sockeye
spec:
  broker: default
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: sockeye
```
