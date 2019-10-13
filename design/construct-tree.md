## RFC: PUBLISH CDK CONSTRUCT TREE

## Overview

This is a strategy for publishing the construct tree which represents a CDK app as a part of the Cloud Assembly that the CDK generates.

**Goal: Expose the `construct tree` for CDK Apps as an artifact of the Cloud Assembly**

This proposal details the motivation behind the goal, requirements, specification of the construct tree, and the proposed design. The scope of the design is focused on defining what the construct tree should include and the approach to produce it as an output.

Prototype: https://github.com/aws/aws-cdk/tree/prototypes/cdk-look

## Background

ALL CDK applications are composed of constructs, which are the basic building blocks of AWS CDK applications. They encapsulate everything that underlying providers such as AWS CloudFormation need to create cloud components.

A construct can represent a single resource, such as an Amazon Simple Storage Service (Amazon S3) bucket, or it can represent a higher-level component consisting of multiple AWS CDK resources. Examples of such components include a worker queue with its associated compute capacity, a cron job with monitoring resources and a dashboard, or even an entire app spanning multiple AWS accounts and regions.

The CDK CLI is the primary mechanism through which developers currently interact with their AWS CDK applications. It supports various operations throughout the lifecycle of application development from from the initialization of a CDK app from a template to the deployment and destruction of the AWS CloudFormation stacks.

## Motivation

Developers author their CDK applications by leveraging higher-level intent based APIs offered through the [AWS Construct Library](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-construct-library.html). They use CLI commands such as `cdk synth` to produce the cloud assembly and `cdk deploy` to deploy it to AWS CloudFormation. When deployments fail, they often have to drop into the AWS CloudFormation templates that the CDK generated or log into the AWS console to trace their issues.

As an example, the following CDK application for an Application Load Balancer includes constructs from Auto Scaling, VPC, and Application Load Balancing.

```typescript
const vpc = new ec2.Vpc(this, 'VPC');

const asg = new autoscaling.AutoScalingGroup(this, 'ASG', {
  vpc,
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T2, ec2.InstanceSize.MICRO),
  machineImage: new ec2.AmazonLinuxImage(),
});

const lb = new elbv2.ApplicationLoadBalancer(this, 'LB', {
  vpc,
  internetFacing: true
});

const listener = lb.addListener('Listener', {
  port: 80,
});

listener.addTargets('Target', {
  port: 80,
  targets: [asg]
});

listener.connections.allowDefaultPortFromAnyIpv4('Open to the world');

asg.scaleOnRequestCount('AModestLoad', {
  targetRequestsPerSecond: 1
});
```

Running `cdk synth` on this application produces ~800 lines of CloudFormation template and represents over 35 AWS resources spanning 5 services. The generated template is part of the cloud assembly and will be used subsequently by deploy and is staged in the `cdk.out` directory. This showcases the power of the CDK in the ability to define high level abstractions of infrastructure in familiar languages that can be reused.

Continuing with this example, inspecting the the cloud assembly’s CloudFormation template doesn’t help gain any insight when behavior does not match expectations.

In this scenario, these are some details that are not intuitive without diving deeper:

>1. What resources does the AutoScalingGroup construct contain? the VPC construct? etc.
>2. What properties were the resources initialized with? Can they be changed?
>3. Is the relationship between these constructs captured in the cloud assembly correctly?


When deployments fail, or provisioned resources don’t hold the properties that one might expect, developers can’t figure out what properties that resources such as `NatGateway, AutoScalingGroup, Subnets, Route Tables` were configured with without dropping into the generated CloudFormation template. It can take a good amount of clicking and digging before developers can figure out what happened and almost certainly needs them to get away from their favourite IDE and switch context to find out.

The recurring theme is that intent based constructs are easy to get started with, take a lot of complexity away from the developer. The flip-side of this experience is that debugging and figuring out what the CDK did on your behalf can become challenging as applications come more complex. This problem will proliferate as more third-party constructs become available for consumption.

 Exposing the construct tree will enrich the developer experience for CDK application construction by exposing the construct tree explicitly.


Let’s take another look at an example of what a construct tree might look like:

This is a Lambda function that has it’s handler code written as an asset:

```typescript
 new lambda.Function(this, 'HelloWorldHandler', {
            runtime: lambda.Runtime.NODE_JS_8_10,
            code: lambda.Code.directory('lambda'),
            handler: 'hello.handler'
 });
```

Ideally this simple Lambda code, will be represented by a construct tree that looks something like the following, with the ability to expand on constructs and navigate all the way down to their properties and values:

```bash
├── App
│   ├── Stack 1
│   │   ├── Lambda Function [L2 Construct](... properties ...)
|   │   │   ├── IAM Role (policy information)
|   │   │   ├── Asset Code (Lambda Handler, pointer to S3 location where Asset will reside)
|   │   │   ├── Lambda Function (Function definition and properties)
```

## Requirements

This list describes only the minimal set of requirements from this feature. After we release these building blocks, we will look into extending the model to meet other use cases that arise.

1. **Format** - Publish underlying data as a `.json` file that contains the information required to render a construct tree view. The construct tree illustrated in the previous section is only a depiction of what a human readable(ish) view might look like
2. **Constructs**
    1. ***Nodes*** - CDK application constructs at the root and drill down to the constructs it contains (1+ cloud resources.) Initially, we will focus our attention on AWS resources, but the model is extensible to cloud components from any provider
    2. **Types/Hierarchy** - Types and hierarchy information will contain the details around the level of abstraction (L1, L2, L3..), service(s), and resource(s) that they represent.
    3. ***Metadata -*** Constructs will expose metadata which will include an array of objects associated with the construct.
        1. **Location** - The full path of the location where the construct is defined. This will support use cases where the construct-tree nodes can support navigability back to the declaration in the IDE itself.
        2. ***Properties*** - Constructs will opt into the set of properties, and these will be exposed. Higher level constructs (L2 and L3) will expose properties and values of the resources they contain and not necessarily enumerate every property that’s available for a resource. This will especially be useful for constructs that are opinionated and set defaults on behalf of developers. Low level constructs (L1) will expose all properties that are configured.
        3. ***Errors/Warnings*** - Errors and warnings that are produced during synthesis indicating validation failures, deprecation notices, guidance, etc will be included in the tree. These are emitted at the construct level although the message may point towards a specific property. The construct tree should be complete and include validation failures for all resources in the CDK application rather than just just fail on the first error.
    4. **Assets** - Assets information will be included in the construct tree. Assets represent actions that the CDK *will* take ahead of stack deployments (i.e. S3 assets are zipped and uploaded directly, Docker images are uploaded to ECR). The asset metadata in the tree is the S3 key that an asset *would* have
3. **Local** - The construct tree is produced at synthesis time. Connectivity to AWS and CloudFormation should not be required to produce a tree view of a CDK application.

## Approach

At a high-level, we will render the `construct tree` through a construct tree data model and construct metadata interface that constructs will implement. This section will describe the components of the JSON structure that will represent the construct-tree:

### Construct Tree Data Model

A feature will be added to the CDK core module so that any CDK application will produce an artifact called `tree.json` during calls to `cdk synth` and include it as an output to the cloud assembly in `cdk.out`.

The construct tree will be a list of paths that are indexed into a map of constructs. The file structure itself represents the tree. This information will need to include the following to start rendering a map of constructs:

### Construct properties

|Property   |Type             |Required     |Source     | Description  |
|---        |---              |---          |---        | ---          |
|id         |string           |Required     |TODO       | id of the construct within the current scope |
|path       |string           |Required     |TODO       | full, absolute path of the construct within the tree |
|children   |Array<Path>      |Not Required |TODO       | All direct children of this construct. Array of the absolute paths of the constructs. Will be used to walk entire list of constructs |
|metadata   |Array<Metadata>  |Not Required |TODO       | Metadata describing all constructs/resources that are encapsulated by the construct  |

### Metadata Properties

The following metadata properties will be included by the construct that produces the `tree.json` output.

|Property       |Type             |Required     |Source | Description    |
|---            |---              |---          |---    | ---            |
|sourceLocation |string           |Required     |TODO   | location in source code where the construct is defined |
|description    |string           |Not Required |TODO   | description of the construct |
|properties     |Array<Metadata>  |Not Required |TODO   | constructs can fill in arbitrary metadata such as configuration. ClouFormation agnostic. CFN constructs could pre-populate it with the output of `this.toCloudFormation()` |
|errors         |Array<Metadata>  |Not Required |TODO   | array of errors associated with this construct.  |
|warnings       |Array<Metadata>  |Not Required |TODO   | array of warnings associated with this construct |

**TODO** - add detail and a more concrete walkthrough with samples of all the attributes referenced in construct and metadata properties.

```json
{
    "version": "1.0-experimental",
    "tree": {
        "id": "App",
        "path": "/",
        "metadata": {
            "sourceLocation": ""
        },
        "children": [
            {
                "id": "CdkWorkshopStack",
                "path": "CdkWorkshopStack",
                "metadata": [
                    {
                        "sourceLocation": "/lib/cdkworkshop-stack.ts:8:22",
                        "description": "Construct for a CloudFormation Stack",
                        "properties": [
                            {
                                "notificationArn": "sample-notification-arn",
                                "retentionPolicy": "RETAIN",
                                "role-arn": "sample-role-arn"
                            }
                        ],
                        "warnings": [
                            {
                                "deprecatedParameter": "`transform` is deprecated, use `transforms` instead"
                            }
                        ]
                    }
                ],
                "children": [
                    {
                        "id": "HelloHandler",
                        "path": "CdkWorkshopStack/HelloHandler",
                        "metadata": [
                            {
                                "sourceLocation": "sample-source-location",
                                "properties": [
                                    {
                                        "memorySize": "128MB",
                                        "Role": "sample-role",
                                        "Tags": [
                                            {
                                                "sample-tag": "yay-cdk",
                                                "safe-to-delete": "yup"
                                            }
                                        ]
                                    }
                                ]
                            }
                        ],
                        "children": [
                            {
                                "id": "ServiceRole",
                                "path": "CdkWorkshopStack/HelloHandler/ServiceRole",
                                "metadata": [
                                    {
                                        "sourceLocation": "sample-source-location",
                                        "properties": "...properties"
                                    },
                                    {
                                        "id": "Resource",
                                        "path": "CdkWorkshopStack/HelloHandler/Resource",
                                        "metadata": [
                                            {
                                                "sourceLocation": "sample-source-location",
                                                "properties": [
                                                    {
                                                        "logicalId": "HelloHandler2E4FBA4D"
                                                    }
                                                ]
                                            }
                                        ]
                                    },
                                    {
                                        "id": "Code",
                                        "path": "CdkWorkshopStack/HelloHandler/Code",
                                        "metadata": [
                                            {
                                                "sourceLocation": "sample-source-location"
                                            }
                                        ],
                                        "children": [
                                            {
                                                "id": "S3Bucket",
                                                "path": "CdkWorkshopStack/HelloHandler/Code/S3Bucket",
                                                "resourecType": "AWS::S3::Bucket",
                                                "metadata": [
                                                    {
                                                        "sourceLocation": "sample-source-location"
                                                    }
                                                ]
                                            },
                                            {
                                                "id": "S3VersionKey",
                                                "path": "CdkWorkshopStack/HelloHandler/Code/S3VersionKey",
                                                "metadata": [
                                                    {
                                                        "sourceLocation": "sample-source-location"
                                                    }
                                                ]
                                            },
                                            {
                                                "id": "AssetBucket",
                                                "path": "CdkWorkshopStack/HelloHandler/Code/AssetBucket",
                                                "metadata": [
                                                    {
                                                        "sourceLocation": "sample-source-location"
                                                    }
                                                ]
                                            }
                                        ]
                                    }
                                ],
                                "children": [
                                    {
                                        "id": "Resource",
                                        "path": "CdkWorkshopStack/HelloHandler/ServiceRole/Resource",
                                        "metadata": [
                                            {
                                                "sourceLocation": "sample-source-location",
                                                "properties": [
                                                    {
                                                        "logicalId": "HelloHandlerServiceRole11EF7C63"
                                                    }
                                                ]
                                            }
                                        ]
                                    }
                                ]
                            }
                        ]
                    }
                ]
            }
        ]
    }
}
```

### Construct metadata

We will also introduce an interface that constructs *can* implement to provide metadata such as properties,
additional context, anything a construct would want to report back as properties.
Constructs will be able to supply information about themselves for inclusion into the context tree.

The context tree provides the skeleton and the structure for rendering a tree-view of a CDK application.
The proposed Interface will be added to `construct.ts` in the core library and implemented by `constructs`
that have additional information to contribute to the construct tree :

```typescript
/***
 * Displayable information that a construct will contribute towards the context tree*
 * such as dependencies, properties, links, documentation, etc
 */
export interface IDisplayable {

  // id of the construct within the current scope
  readonly displayName: string

  // Other displayable attributes
  readonly attributes: [key: string]: string;

  /* method that supports extensibility on what can be displayed without
   * adding more methods/properties on IDisplayable
   */
  inspect(display: IDisplayer): void
}
```

## Unresolved Questions

* Experience across all supported languages - nuances, considerations, etc

## Future Possibilities

We do not expect customers to have to interact directly with the outputs that the CDK produces as a part of the cloud assembly. Publishing the CDK construct tree as an output to the cloud assembly is intended to be a starting point to build on and create experiences based on it.

### Tooling

With a construct tree model in place and construct tree being published as a `.json` file, it opens the door to build tooling and utilities to enhance the application development experience so that developers can gain context and insight without having to leave their IDEs. Some of these features could include:

**Diff** diff the construct tree against the resources that are deployed in the cloud. This capability would require cloud capability but we should always be able to render a construct tree without the need to connect.

**Navigability** include the location in code where the construct is defined. This would start to pave the way towards building tooling and extensions that IDE's can leverage.