# Amazon SNS target

Sends event to [Amazon SNS](https://aws.amazon.com/sns/).

With `tmctl`:

```
tmctl create target awssns --arn <arn> --auth.credentials.accessKeyID <access key> --auth.credentials.secretAccessKey <secret key>
```

On Kubernetes:

Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "<AWS Access Key ID>"
  AWS_SECRET_ACCESS_KEY: "<AWS Secret Access Key>"
```

Target

```yaml
apiVersion: targets.triggermesh.io/v1alpha1
kind: AWSSNSTarget
metadata:
  name: triggermesh-aws-sns
spec:
  arn: arn:aws:sns:us-west-2:<PROJECT_ID>:tmtest
  auth:
    credentials:
      accessKeyID:
        valueFromSecret:
          name: aws
          key: AWS_ACCESS_KEY_ID
      secretAccessKey:
        valueFromSecret:
          name: aws
          key: AWS_SECRET_ACCESS_KEY
```

Alternatively you can use an IAM role for authentication instead of an access key and secret, for Amazon EKS only:

```yaml
auth:
  iamrole: arn:aws:iam::123456789012:role/foo
```

To setup an IAM role for service accounts, please refer to the [official AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html).

There is an optional toggle flag indicating if the full CloudEvent should be sent
to SNS. By default, this is disabled which means only the event payload
will be sent.

Accepts events of any type.

Responds with events with the following attributes:

* type `io.triggermesh.targets.aws.sns.result`
* source `arn:aws:sns:...`, the SNS ARN value as configured by the target

See the [Kubernetes object reference](../../reference/targets/#targets.triggermesh.io/v1alpha1.AWSSNSTarget) for more details.

## Prerequisite(s)

- AWS API key and secret
- ARN for the SNS topic to invoke

For more information about using AWS Simple Notification Service, please refer to the [AWS documentation][docs].

[ce]: https://cloudevents.io/
[docs]: https://docs.aws.amazon.com/sns/
[ce-jsonformat]: https://github.com/cloudevents/spec/blob/v1.0/json-format.md
