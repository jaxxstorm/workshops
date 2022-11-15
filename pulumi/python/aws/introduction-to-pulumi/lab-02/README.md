# Creating your first resource

Pulumi creates resources and tracks them inside its _state_. Any resource declared in your Pulumi program will be declaratively created by Pulumi

## Create an S3 bucket

We'll create an S3 bucket that is managed by Pulumi. Add the following to your `__main__.py`

```python
import pulumi_aws as aws

bucket = aws.s3.Bucket('my-bucket')
```

Run `pulumi up` from your terminal. You'll see some output that indicates your bucket has been created

## Observe the name of the bucket

We'll observe Pulumi's ability to output properties of the bucket. Add the following to your `__main__.py`

```python
pulumi.export("bucket_name", bucket.bucket)
```

Run `pulumi up` again.

### Autonaming

We are using Pulumi's stack outputs to examine the _output property_ of a Pulumi resource.

You'll notice that the bucket name of the created bucket doesn't match the resource name. Pulumi autonames resources for you so that you can easily manage resources in the cloud.

## Change the name of the bucket

Update your Pulumi code to modify the resource name:

```
import pulumi_aws as aws

bucket = aws.s3.Bucket('the-bucket')
```

Run `pulumi up` again. You'll notice that because the name of the bucket has changed, Pulumi is going to replace the resource. The use of autonaming means that Pulumi can replace the resource easily without running into API naming conflicts.

## Add tags to the bucket

Finally modify the bucket to add some tags. Update your `__main__.py` like so:

```python
bucket = aws.s3.Bucket(
    "the-bucket",
    tags={
        "Owner": "lbriggs",
    }
)
```

Run `pulumi up` and note the difference: Pulumi can modify the tags on the bucket without needing to create a new bucket, so it modify the bucket in place.

