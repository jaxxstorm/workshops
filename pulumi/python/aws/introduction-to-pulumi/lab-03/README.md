# Inputs & Outputs

Pulumi uses an asynchronous input and out mechanism to allow it to build a resource graph. In this lab, we'll examine how that works and operates

## Create some bucket objects

We want to upload some files to our S3 bucket. Create a directory called `www` and create two files within it:

- `index.html` - Add some HTML to this, like `<p>Hello world!</p>`
- `404.html` - Add some HTML, like `<p>This is an error</p>`

## Add bucket object 

We'll now want to upload these objects into the bucket. We can evaluate all the files in the folder and create a new Pulumi resource for each, using standard python libraries.

Update your `__main.py__` to add the following code.

```python
import mimetypes
import os

content_dir = "www"

"""
Loop through the contents of the www folder
and upload it to the bucket as a bucket object
"""
for file in os.listdir(content_dir):
    filepath = os.path.join(content_dir, file)
    mime_type, _ = mimetypes.guess_type(filepath)
    obj = aws.s3.BucketObject(
        file,
        bucket=bucket.id,
        source=pulumi.FileAsset(filepath),
        content_type=mime_type,
    )
```

Run your pulumi up and notice how Pulumi wants to create new objects for each file in your `www` directory.

There are some things to note here:

### Order of Operations

You'll notice that the bucket objects are created after the bucket is created.

This is because the objects are passed the `bucket.id` output. This tells Pulumi that the `bucket.id` must be resolved (ie: returned from the API) before it can create the bucket objects.

This input/output system is how Pulumi builds is a declarative graph and tracks dependencies.

### Usage of standard python packages

We can detect the `mime_type` using the standard mimetypes package from Python. This flexibility is a result of using the Python language

### Loops

We loop through the directory containing the objects using a standard Python for loop.

## Create a bucket policy

Now we have a bucket and some bucket objects, we need to create a bucket policy.

We want our bucket policy to only apply to the bucket we're creating, so we need to interpolate the arn of the bucket that was created into the policy object.

Unfortunately, because the `bucket.arn` is a value returned from the AWS API, we can't directly interpolate the value into a JSON python string for the policy object. We need to _resolve_ this and wait for it to be available first.

In Pulumi, we use `apply` for this.

Add the following to your `__main__.py`:

```python
import json

"""
Set a bucket policy on our bucket that allows public get object
Notice the use of apply to populate the policy
"""
bucket_policy = aws.s3.BucketPolicy(
    "my-bucket-policy",
    bucket=bucket.id,
    policy=bucket.arn.apply(
        lambda arn: json.dumps({
            "Version": "2012-10-17",
            "Statement": [{
                "Effect": "Allow",
                "Principal": "*"
                "Action": [
                    "s3:GetObject"
                ],
                "Resource": [
                    f"{arn}/*"
                ]
            }]
        })),
    opts=pulumi.ResourceOptions(parent=bucket)
)
```
Note the `bucket.arn.apply` statement here. Once we are inside this `apply` block, we can use the arn in the `json.dumps` function as we'd expect to with any other string.

## Configure Website

The final step here is to make our S3 bucket have some website configuration, so it works as a static website. Modify the bucket configuration to indicate you wish to use it as a website, like so:

```python
bucket = aws.s3.Bucket(
    "the-bucket",
    tags={
        "Owner": "lbriggs",
    },
    acl="public-read",
    website=aws.s3.BucketWebsiteArgs(
        index_document="index.html",
        error_document="404.html",
    )
)
```

You'll notice here we're using strongly type arguments as part of Python's type system. We can then export the website endpoint like so:

```python
pulumi.export("bucket_endpoint", bucket.website_endpoint)
```

Run `pulumi up` to finish your provisioning. Observe the bucket we have created, the objects and the policy associated. We can also navigate to the website:



Then run `pulumi destroy` to destroy the resources
