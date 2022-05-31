# Creating a Bridge With a Dead Letter Sink (DLS)

## What is a Dead Letter Sink?

TriggerMesh provides various configuration parameters to control the delivery of events in case of failure. For instance, you can decide to retry sending events that failed to be consumed, and if this didn't work you can decide to forward those events to a dead letter sink.

## Implementing a Dead Letter Sink

There are two ways to implement a dead letter sink, either by use of a Subscription or by use of a Broker.

### Subscription Configuration

You can configure how events are delivered for each Subscription by adding a delivery spec to the Subscription object, as shown in the following example:

```yaml
apiVersion: messaging.knative.dev/v1
kind: Subscription
metadata:
  name: example-subscription
  namespace: example-namespace
spec:
  delivery:
    deadLetterSink:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: example-sink
    backoffDelay: <duration>
    backoffPolicy: <policy-type>
    retry: <integer>
```

Where: 

The `deadLetterSink` spec contains configuration settings to enable using a dead letter sink. This tells the Subscription what happens to events that cannot be delivered to the subscriber. When this is configured, events that fail to be delivered are sent to the dead letter sink destination. The destination can be a Knative Service or a URI. In the example, the destination is a Service object, or Knative Service, named example-sink.

The `backoffDelay` delivery parameter specifies the time delay before an event delivery retry is attempted after a failure. The duration of the backoffDelay parameter is specified using the ISO 8601 format. For example, PT1S specifies a 1 second delay.

The `backoffPolicy` delivery parameter can be used to specify the retry back off policy. The policy can be specified as either linear or exponential. When using the linear back off policy, the back off delay is the time interval specified between retries. When using the exponential back off policy, the back off delay is equal to backoffDelay*2^<numberOfRetries>.
retry specifies the number of times that event delivery is retried before the event is sent to the dead letter sink.

  
### Broker Configuration

You can configure how events are delivered for each Broker by adding a delivery spec, as shown in the following example:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: with-dead-letter-sink
spec:
  delivery:
    deadLetterSink:
      ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: example-sink
    backoffDelay: <duration>
    backoffPolicy: <policy-type>
    retry: <integer>
```

### Example Broker Configuration

Lets take a look at an example of a Broker with a dead letter sink and configure a simple bridge with a dead letter sink. For our discussion, let us consider [this example Bridge](../assets/yamlexamples/simple-bridge.yaml) as a starting point:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: events
---
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: ping-sockeye
spec:
  data: '{"name": "triggermesh"}'
  schedule: "*/1 * * * *"
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: events
```
  
Now, lets modify this example to add a dead letter sink to the `Broker`. We can accomplish this in two steps:

1. Add a service to catch the "dead letter" events. (We will be using `Sockeye` here, but in a production scenario you would want to use something like SQS, Kafka, or another Sink that has some form of persistence.)

```yaml
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
```

2. Update the `Broker` section to the following:

```yaml
---
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
name: events
spec:
  delivery:
    deadLetterSink:
    ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: sockeye
    backoffDelay: "PT0.5S"     # or ISO8601 duration
    backoffPolicy: exponential # or linear
    retry: 2
```

## Viewing the Results of a Dead Letter Sink

Now that we have all the parts in place, we can monitor the events that are being sent to the dead letter sink. We can do this in two ways:

1. View the pod logs of the `sockeye` service:
  * `kubectl get pods` will show the pods that are running. Retrieve the sockeye pod name from the output.
  * `kubectl logs <SOCKEYE_POD_NAME> user-container` By replacing the `<SOCKEYE_POD_NAME>` with the pod name you can view the logs of the sockeye pod.

2. View the web service exposed by the `sockeye` service:
  * `kubectl get ksvc` will show the KSVC's that are running. Retrieve the sockeye public URL from the `URL` column and navigate to it in your browser.