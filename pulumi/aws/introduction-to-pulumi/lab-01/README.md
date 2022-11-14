# Creating a New Project

Pulumi uses projects to store code, configuration and stacks.

## Prerequisites

Before you begin, you should verify you're logged into a Pulumi backend:

```
pulumi whoami -v
```

## Getting Started

To start building Pulumi projects and code, you first need to create a new project.

Create a new directory in your favourite place to store code:

```
mkdir pulumi-training-<yourname>
```

You can do this using the `pulumi new` command:

```
pulumi new python
```

## Adding a Provider

Providers allow you to interact with cloud provider APIs. In this workshop, we'll interact with the AWS provider

Install the provider inside the Pulumi virtual env


```
venv/bin/pip3 install pulumi_aws
```

This will install the AWS SDK and the AWS provider plugin. You can verify it was installed using

```
pulumi plugin ls
```

## Add some configuration to your stack

Every Pulumi provider has the ability to read _configuration_ which is specific to a stack.

We want to select the AWS region we're going to use to provision resources. We can do this using stack configuration.

Set the region in your created stack like so:

```
pulumi config set aws:region us-west-2
```
