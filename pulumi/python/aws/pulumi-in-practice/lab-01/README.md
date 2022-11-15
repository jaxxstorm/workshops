# EC2 Component

In this lab, we'll create a Component called a webserver.

## Step 1 &mdash; Define an empty component

Create a new Pulumi project. Inside that new project, create a new file, webserver.py

Here, we'll create an empty component called webserver:

```python
# webserver.py
# An example component
webserv
```

Our webserver has two classes, one which contains the actual webserver itself, and one for its args. 


Inside the `__main__.py` we now import this class

```python
# __main__.py
from webserver import WebServer, WebServerArgs

webserver = WebServer("lbriggs", WebServerArgs(instance_type="t3.micro"))
```

We can run `pulumi up` here and see our empty webserver defined:

```bash
     Type                     Name                                Plan
 +   pulumi:pulumi:Stack      pulumi-aws-webserver-component-dev  create
 +   └─ custom:app:WebServer  lbriggs                             create
```

## Step 2 &mdash; Add a basic resource to our webserver

Now inside your component resource, let's add some simple resources.

```python
# webserver.py
import pulumi
import pulumi_aws as aws


class WebServerArgs:
    def __init__(
        self,
        instance_type: pulumi.Input[str],
        vpc_id: str,
    ):
        self.instance_type = instance_type
        self.vpc_id = vpc_id


class WebServer(pulumi.ComponentResource):

    security_group: aws.ec2.SecurityGroup

    def __init__(
        self, name: str, args: WebServerArgs, opts: pulumi.ResourceOptions = None
    ):
        super().__init__("custom:app:WebServer", name, {}, opts)

        # create a security group
        self.security_group = aws.ec2.SecurityGroup(
            f"{name}-securitygroup",
            vpc_id=args.vpc_id,
            ingress=[
                {
                    "protocol": "tcp",
                    "from_port": 80,
                    "to_port": 80,
                    "cidr_blocks": ["0.0.0.0/0"],
                }
            ],
            egress=[
                {
                    "protocol": "tcp",
                    "from_port": 80,
                    "to_port": 80,
                    "cidr_blocks": ["0.0.0.0/0"],
                }
            ],
            opts=pulumi.ResourceOptions(parent=self),
        )
```

There are several things to note here:

### Class attributes

Class attributes allow us to access the properties of the component later.

### Resource parent

We set the `opts=pulumi.ResourceOptions(parent=self)` property to ensure our security group is "parented" to the component. This means it inherits all the other properties.

We now need to update the `__main__.py` to ensure we're passing a vpc id to our component.

We're going to look up the default VPC using a `get` call. This allows us to retrieve information for resources we haven't provisioned using Pulumi.

```python
from webserver import WebServer, WebServerArgs
import pulumi_aws as aws

vpc = aws.ec2.get_vpc(default=True)

server = WebServer("lbriggs", WebServerArgs(instance_type="t3.micro", vpc_id=vpc.id))
```

