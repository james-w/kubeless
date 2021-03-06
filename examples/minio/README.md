# Minio Example

This example uses the Minio S3 clone to show case a kubeless pubsub function.
Minio is configured to send notifications to Kafka, a function consumes those events and performs actions.

The Minio access keys are stored as a Kubernetes secret

## Deploy Minio via Helm

If you have cloned the kubeless repo:

```
cd examples
cd minio
helm install ./minio
```

This will use the default values in `./minio/values.yaml`

Check the logs of the Minio pod for configuration info.

## Minio configuration

You will need the Minio [client](https://github.com/minio/mc) `mc`:

```
brew install minio-mc
```

The following are Minio specific, it assumes your are using minikube on `192.168.99.100`

```
mc config host add localminio http://192.168.99.100:32751 foobar foobarfoo
```

Login to the Minio UI and create a `foobar` and a `ocr` bucket:

```
minikube services <helm_release_name_for_minio>
```

Turn on events for a `foobar` bucket:

```
mc events add localminio/foobar arn:minio:sqs:us-east-1:1:kafka
```

Check bucket

```
mc ls localminio/<bucket_name>
```

## Use Kubeless to echo Minio events

Write a echo function

```
def handler(context):
    return context
```

Create a topic (by default the Helm chart used the `s3` topic):

```
kubeless topic create s3
```

Create the function with kubeless:

```
kubeless function create pubsub --trigger-topic s3 --runtime python27 --handler pubsub.handler --from-file python/pubsub.py
```

Watch the logs of the pods for Minio events:

```
$ kubectl logs -f pubsub-2287267129-07czk
{u'level': u'info', u'EventType': u's3:ObjectRemoved:Delete', u'Records': [{u'eventVersion': u'2.0', u'eventTime': u'2017-03-17T10:52:30Z', u'requestParameters': {u'sourceIPAddress': u'172.17.0.1:57970'}, u's3': {u'configurationId': u'Config', u'object': {u'sequencer': u'14ACA5DE4CE37B2F', u'key': u'foobar.py'}, u'bucket': {u'arn': u'arn:aws:s3:::foobar', u'name': u'foobar', u'ownerIdentity': {u'principalId': u'foobar'}}, u's3SchemaVersion': u'1.0'}, u'responseElements': {u'x-amz-request-id': u'14ACA5DE4CE37B2F', u'x-minio-origin-endpoint': u'http://172.17.0.9:9000'}, u'awsRegion': u'us-east-1', u'eventName': u's3:ObjectRemoved:Delete', u'userIdentity': {u'principalId': u'foobar'}, u'eventSource': u'aws:s3'}], u'Key': u'foobar/foobar.py', u'time': u'2017-03-17T10:52:30Z', u'msg': u''}
<type 'dict'>
{u'level': u'info', u'EventType': u's3:ObjectCreated:Put', u'Records': [{u'eventVersion': u'2.0', u'eventTime': u'2017-03-17T10:52:42Z', u'requestParameters': {u'sourceIPAddress': u'172.17.0.1:57970'}, u's3': {u'configurationId': u'Config', u'object': {u'eTag': u'd8e9cdac05040c688fe2ea331bdc97fd', u'sequencer': u'14ACA5E103E5C379', u'key': u'cli.py', u'size': 177}, u'bucket': {u'arn': u'arn:aws:s3:::foobar', u'name': u'foobar', u'ownerIdentity': {u'principalId': u'foobar'}}, u's3SchemaVersion': u'1.0'}, u'responseElements': {u'x-amz-request-id': u'14ACA5E103E5C379', u'x-minio-origin-endpoint': u'http://172.17.0.9:9000'}, u'awsRegion': u'us-east-1', u'eventName': u's3:ObjectCreated:Put', u'userIdentity': {u'principalId': u'foobar'}, u'eventSource': u'aws:s3'}], u'Key': u'foobar/cli.py', u'time': u'2017-03-17T10:52:42Z', u'msg': u''}
<type 'dict'>
```

## Use Kubeless for managing files

Your function will need access to Minio, create a secret with your access keys:

```
kubectl create secret generic minio --from-literal=access_key=foobar --from-literal=secret_key=foobarfoo
```

Create the function:

```
kubeless function create minio --trigger-topic s3 --runtime python27 --handler minio-test.ocr --from-file minio-test.py --dependencies requirements.txt
```

Create a `foobar` and a `ocr` bucket in Minio UI.
Add an object in the `foobar` bucket and watch it being put in the `ocr` bucket with `.ocr` extension
