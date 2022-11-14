# Attach our web servers to our load balancer

## Step 1 &mdash; Add a target group

We'll now add a target group to the component. We aren't going to associate the target group to a class instance in this case because we don't need to access the target group outside our component.

```python
# add a target group to the LoadBalancer
target_group = aws.lb.TargetGroup(
 	f"{name}-targetgroup",
	port=80,
	protocol="HTTP",
	target_type="ip",
	vpc_id=vpc.id,
	opts=pulumi.ResourceOptions(parent=self)
)
```

## Step 2 &mdash; Add a listener

Now we'll define an ALB listener that listens on port 80

```python
# define a listener
listener = aws.lb.Listener(
	f"{name}-listener",
	load_balancer_arn=lb.arn,
	port=80,
	default_actions=[
		{
		    "type": "forward",
			"target_group_arn": target_group.arn,
		}
	],
	opts=pulumi.ResourceOptions(parent=lb),
)
```

## Step 3 &mdash; Add the attachments

Now we've defined the listener and the target group, we need to define a target group attachment for each EC2 instance.

We do this inside the `enumerate` loop, adding the following code:

```python
# create a loadbalancer attachment for each server
attachment = aws.lb.TargetGroupAttachment(
    f"{name}-webserver-{i}",
	target_group_arn=target_group.arn,
	target_id=server.private_ip,
	port=80,
	opts=pulumi.ResourceOptions(parent=target_group)
)
```

The final code for our webserver looks like this

```python
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
    servers: list[aws.ec2.Instance] = []
    lb: aws.lb.LoadBalancer

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

        self.lb = aws.lb.LoadBalancer(
            f"{name}-lb",
            security_groups=[self.security_group.id],
            subnets=args.subnet_ids,
            load_balancer_type="application",
            opts=pulumi.ResourceOptions(parent=self),
        )

        # add a target group to the LoadBalancer
        target_group = aws.lb.TargetGroup(
            f"{name}-targetgroup",
            port=80,
            protocol="HTTP",
            target_type="ip",
            vpc_id=args.vpc_id,
            opts=pulumi.ResourceOptions(parent=self),
        )

        # define a listener
        listener = aws.lb.Listener(
            f"{name}-listener",
            load_balancer_arn=self.lb.arn,
            port=80,
            default_actions=[
                {
                    "type": "forward",
                    "target_group_arn": target_group.arn,
                }
            ],
            opts=pulumi.ResourceOptions(parent=self.lb),
        )

        user_data = '#!/bin/bash echo "Hello, World!" > index.html nohup python3 -m SimpleHTTPServer 80 &'

        for i, subnet_name in enumerate(args.subnet_ids, start=0):
            server = aws.ec2.Instance(
                f"{name}-webserver-{i}",
                instance_type=args.instance_type,
                vpc_security_group_ids=[self.security_group.id],
                ami=args.ami_id,
                subnet_id=subnet_name,
                user_data=user_data,
                opts=pulumi.ResourceOptions(parent=self),
            )
            self.servers.append(server)

            attachment = aws.lb.TargetGroupAttachment(
                f"{name}-webserver-{i}",
                target_group_arn=target_group.arn,
                target_id=server.private_ip,
                port=80,
                opts=pulumi.ResourceOptions(parent=target_group),
            )
```

In your `__main__.py` we can access the name of the loadbalancer to access our webserver. The final code for the `__main__.py` looks like this

```python
from webserver import WebServer, WebServerArgs
import pulumi_aws as aws
import pulumi

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

pulumi.export("lb_dns_name", webserver.lb.dns_name)
```

Run your `pulumi up` to ensure all resources are provisioned, then be sure you delete them with `pulumi destroy`


