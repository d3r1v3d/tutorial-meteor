# Tutorial: Running a Meteor app on AWS using coldbrew-cli

This is a sample project to demonstrate how to run a [Meteor](https://www.meteor.com/) application on AWS using [coldbrew-cli](https://github.com/coldbrewcloud/coldbrew-cli).

## Getting Started

### Install Docker

[coldbrew-cli](https://github.com/coldbrewcloud/coldbrew-cli) deploys your application in [Docker containers](https://www.docker.com/what-docker). So the first step is to [install Docker](https://docs.docker.com/engine/installation/) in your system if you don't have it yet.

### Install coldbrew-cli

Download the package for your system [here](https://github.com/coldbrewcloud/coldbrew-cli/wiki/Downloads). It makes things easier if you copy the downloaded binary `coldbrew` (or `coldbrew.exe` on Windows) in your `$PATH`, so you can run `coldbrew` from anywhere. _But it's also okay to keep `coldbrew` executable in your application directory if you want._

### AWS Account

As you will be deploying your application on AWS, you will need an AWS account for sure. [Sign up](https://aws.amazon.com/) if you haven't yet, and get your [AWS access keys](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSGettingStartedGuide/AWSCredentials.html). You can pass AWS access keys to **coldbrew-cli** either in [environment variables](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Environment-Variables) or using [CLI flags](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Global-Flags), but we will assume that you set the follow environment variables throughout the turorial:

- `$AWS_ACCESS_KEY_ID`: AWS Access Key ID
- `$AWS_SECRET_ACCESS_KEY`: AWS Secret Access Key
- `$AWS_REGION`: AWS region name
- `$AWS_VPC`: AWS VPC ID _(this is completely optional. If you don't specify or don't know your VPC ID, **coldbrew-cli** will automatically use the [default VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/default-vpc.html) of your AWS account.)_

### Meteor

Pretty obvious, but, you will need to [install Meteor](https://www.meteor.com/install) to follow this tutorial.

### Clone This Repo

This tutorial project contains the bare minimum _(but fully functional)_ sample resources so you can get started right away.

- A sample Meteor application: created by running `meteor create tutorial-meteor`
- [deploy](https://github.com/coldbrewcloud/tutorial-meteor/blob/master/deploy) directory: all deploy related resources including deploy configuration, Dockerfile, and bunch of scripts.
- [coldbrew.conf](https://github.com/coldbrewcloud/tutorial-meteor/blob/master/Makefile): deploy configuration file
- [Makefile](https://github.com/coldbrewcloud/tutorial-meteor/blob/master/Makefile): for Meteor app bundling (`make bundle`) and deployment (`make deploy`)

_Basically, everything should be the regular Meteor app directory structure except for the `deploy` directory, `coldbrew.conf`, and, `Makefile`._

Clone this repo:

```bash
git clone https://github.com/coldbrewcloud/tutorial-meteor.git
cd tutorial-meteor
```

## Creating Your First Cluster

**coldbrew-cli** has _very_ simple [concepts](https://github.com/coldbrewcloud/coldbrew-cli/wiki/Concepts): clusters and applications (apps). An app is the minimum deployment unit and _typically_ corresponds to a project (like this tutorial project). And a cluster is simply a collection of apps who will share some AWS resources. _Most importantly they share the Docker hosts (which is ECS Container Instances in the AWS context)._

Let's create your first cluster `tutorial` using [cluster-create](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-cluster-create) command:

```bash
coldbrew cluster-create tutorial --disable-keypair
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-cluster-create.gif?v=1" width="800">

_*In this tutorial, we used `--disable-keypair` flag to skip assigning EC2 key pairs to the container instances. If you will need direct access to the instances (e.g. via SSH), you can use the `--key` flag to specify your key pair name._

If you want to check the current running status of your first cluster, you can use [cluster-status](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-cluster-status) command:

```bash
coldbrew cluster-status tutorial
```

It can take several minutes until the initial ECS Container Instances (EC2 Instances) become fully available, but, you can continue on to start deploying your application.

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-cluster-status.gif?v=1" width="800">

## Deploying Your Application

Now it's time to deploy your app for the first time. All you need is an application [configuration file](https://github.com/coldbrewcloud/coldbrew-cli/wiki/Configuration-File) to define your app's deployment settings. You can easily create it on your own, or you can use [init](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-init) command to generate a proper default configuration for your app. In this tutorial, we provide a sample [coldbrew.conf](https://github.com/coldbrewcloud/tutorial-meteor/blob/master/coldbrew.conf) file so we can start quickly.

In this tutorial, we will use `make deploy` command to script the typical Meteor app deployment process:

- Create a temporary build directory (`/tmp/tutorial-meteor`)
- Run `meteor build` to compile, build, and, bundle everything in the temporary build directory (`/tmp/tutorial-meteor/bundle`)
- Copy deployment files to the temporary build directory
- Run `coldbrew deploy` to initiate the deployment process

See [Makefile](https://github.com/coldbrewcloud/tutorial-meteor/blob/master/Makefile) if you want more details, and feel free to customize the scripts if you want.

Now let's run it:

```bash
make deploy
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-make-deploy.gif?v=1" width="800">

If you're curious, this is what happens when **coldbrew-cli** initiates the deployment process:

- **coldbrew-cli** builds local Docker image using [Dockerfile](https://github.com/coldbrewcloud/tutorial-meteor/blob/master/deploy/Dockerfile). _(You can skip this if you provide the local image name using `--docker-image` flag directly.)_
- It pushes the Docker image to the ECR Repository that's created for your application.
- It creates, updates, or configures ECS Task Definition and ECS Service to apply your configurations.
- It also creates or configures an ELB Application Load Balanacer if your application needs a load balancer. _(This tutorial enables the load balancer.)_

Now let's check the application status using [status](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-status) command:

```bash
coldbrew status
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-status.gif?v=1" width="800">

It gives you much more details about your application and its related AWS resources.

_*Again, it will take several minutes until all AWS resources get fully provisioned and become active (especially if you enabled load balancer). But, the next deploys will be much faster, typically within a minute._

#### Deploying Again

After the first deploy, whenever you have changes in your code or configurations, you can re-deploy them again using the same [deploy](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-deploy) command. In this example, I made a simple change in `.units` attributes in the configuration file: from `1` tom `2`.

```bash
make deploy
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-deploy-2.gif?v=1" width="800">

This time, you will notice that **coldbrew-cli** did not create a new AWS resources this time because they were already created during the first deploy run. **coldbrew-cli** always tries to minimize the actual AWS changes by analyzing and comparing the current status and the desired status.

_*Note that not all configuration changes will be applied immediately. See [this](https://github.com/coldbrewcloud/coldbrew-cli/wiki/Configuration-Changes-and-Their-Effects) for more details._

Let's run [status](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-status) command again:

```bash
coldbrew status
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-status-2.gif?v=2" width="800">

## Testing the Application

To know if your application is up and running, check [status](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-status) command output:
- Make sure ELB Load Balancer's Status becomes `active`.
- Make sure you have more than 1 ECS Task has running status (`RUNNING/RUNNING`).

Assuming it's all good, let's test if your Meteor application really works. Again check [status](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-status) command output to find the Load Balancer endpoint _(`http://tutorial-meteor-elb-352619032.us-west-2.elb.amazonaws.com:80` in my example run)_. You should be able to test your app by opening the load balancer endpoint URL in your web browser.

## Cleaning Up

When you no longer need to run your application in AWS, you can use the [delete](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-delete) command to clean up all AWS resources that were used to run your application.

```bash
coldbrew delete
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-meteor-delete.gif?v=1" width="800">

_*Note that cleanining up can take several minutes to finish to make sure all AWS resources are properly deleted or updated._

And, if you need to delete the cluster too, you can [cluster-delete](https://github.com/coldbrewcloud/coldbrew-cli/wiki/CLI-Command:-cluster-delete) command. 

```bash
coldbrew cluster-delete tutorial
```

<img src="https://raw.githubusercontent.com/coldbrewcloud/assets/master/coldbrew-cli/tutorial-nodejs-cluster-delete.gif?v=1" width="800">

_*For the same reason, cluster delete can take long to finish._

## Notes

- In this tutorial, we didn't use `$MONGO_URL`, but, in production you should set up your own MongoDB cluster (one easy way to do is to use [compose.io](https://compose.io)) and specify its connection URI in `$MONGO_URL`. You can use `env` attribute in the configuration file.

---

That's it for the tutorial. Now you know how to run your Meteor applications on AWS using [coldbrew-cli](https://github.com/coldbrewcloud/coldbrew-cli). Although this tutorial used a sample Meteor application, the deployment workflow of **coldbrew-cli** for other tech stacks is almost the same, as long as you have your Dockerfile in place.

See [Documentations](https://github.com/coldbrewcloud/coldbrew-cli/wiki) for more details.
