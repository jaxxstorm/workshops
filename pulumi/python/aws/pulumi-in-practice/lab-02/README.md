# Add the web resources to our component

## Step 1 &mdash; Add our webservers

At this point, we only have a single security group. We probably need to add more resources.

Let's add some ec2 instances to our WebServer component. We want our webserver to be highly available, so let's add one to each subnet we have.

We'll loop through the subnets we retrieved earlier and create a webserver in each, passing the subnet id to the webserver when we create it.

We'll also need to retrieve a valid AMI ID and pass that to our webserver.

```python
# webserver.py

import pulumi
import pulumi_aws as aws

class WebServerArgs:
    def __init__(
        self,
        instance_type: pulumi.Input[str],
        vpc_id: str,
        subnet_ids: list[str],
        ami_id: str,
    ):
        self.instance_type = instance_type
        self.vpc_id = vpc_id
        self.subnet_ids = subnet_ids
        self.ami_id = ami_id


class WebServer(pulumi.ComponentResource):

    security_group: aws.ec2.SecurityGroup
    servers: list[aws.ec2.Instance]

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

        self.servers = []

        for i, subnet_name in enumerate(args.subnet_ids, start=0):
            server = aws.ec2.Instance(
                f"{name}-webserver-{i}",
                instance_type=args.instance_type,
                vpc_security_group_ids=[self.security_group.id],
                ami=args.ami_id,
                subnet_id=subnet_name,
                opts=pulumi.ResourceOptions(parent=self),
            )
            self.servers.append(server)
```

We have added some more argument to our class, so we'll need to add these input arguments to our instantiation of the webserver. Update your `__main__.py` to look like this:

```python
from webserver import WebServer, WebServerArgs
import pulumi_aws as aws

vpc = aws.ec2.get_vpc(default=True)
subnets = aws.ec2.get_subnets(filters=[{"name": "vpc-id", "values": [vpc.id]}])

ami = aws.ec2.get_ami(
    owners=["amazon"],
    most_recent=True,
    filters=[
        aws.ec2.GetAmiFilterArgs(
            name="name",
            values=["amzn2-ami-hvm-2.0.*-x86_64-gp2"],
        )
    ],
)

webserver = WebServer(
    "lbriggs",
    WebServerArgs(
        instance_type="t3.micro", vpc_id=vpc.id, subnet_ids=subnets.ids, ami_id=ami.id
    ),
)
```

## Step 2 &mdash; Add a LoadBalancer

Now we have webserver, let's add a load balancer to balance the requests to them.

We'll use an AWS ALB for this.

Add the following resource to your class:

```python
self.lb = aws.lb.LoadBalancer(
            f"{name}-lb",
			security_groups=[self.security_group.id],
			subnets=args.subnet_ids,
			load_balancer_type="application",
			opts=pulumi.ResourceOptions(parent=self)
        )
```

Ensure you add a class property to assign the load balancer.

At this stage, run `pulumi up` and create the resources in your web server component.
