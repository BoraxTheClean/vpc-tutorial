# VPC Tutorial

![vpc-diagram](https://raw.githubusercontent.com/BoraxTheClean/vpc-tutorial/master/VPCTutorial.png)
![CI Status](https://github.com/BoraxTheClean/vpc-tutorial/workflows/Cloudformation%20CI/badge.svg)

This is a tutorial of how to build a VPC in AWS.

The following is an overview of the components and sub-components of a VPC. We will be using Cloudformation to illustrate and deploy.

[One click deploy link, no AWS charges will be incurred.](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://owen-public-production-bucket.s3.amazonaws.com/vpc-demo/vpc/vpc.template.yaml)


## What is a VPC?

_Amazon Virtual Private Cloud (Amazon VPC) lets you provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define._

### High Level VPC Overview

```bash
vpc
├── subnet
│   ├── route-table
│   │   └── route-table.template.yaml
│   └── subnet.template.yaml
└── vpc.template.yaml
```

  - VPCs have subnets.
  - Subnets have route tables.
  - Route tables have routes.
  - Routes have destinations and targets.

## VPC Code

In order to create a VPC, all we need to do is assign it a CidrBlock. [It is a best practice to assign a private Cidr range to your VPC.](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses)

Resources deployed in this VPC will draw a Private IP address from the Cidr we assigned to our VPC.

```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: !Ref CidrBlock
    Tags:
      - Key: Name
        Value: MyVPC
```

Our VPC will have internet access. So we need an InternetGateway (IGW) and we associate it to our VPC. Later, we will use this IGW to give resources in our VPC access to the internet.

```yaml
  IGW:
    Type: AWS::EC2::InternetGateway

  IGWAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref IGW
      VpcId: !Ref VPC
```

## Subnet Code

VPCs need subnets. Subnets are where actual Amazon resources will be deployed.

Some general notes on Subnets:
  - Subnets exist in a single Availability Zone.
  - Subnets are bound to a single VPC.
  - Subnets have a CidrBlock that is a subset of the VPC's CidrBlock.
  - Subnets have Route Tables associated with them.

```yaml
SubnetOne:
  Type: AWS::CloudFormation::Stack
  Properties:
    Parameters:
      VPC: !Ref VPC
      IGW: !Ref IGW
      AZ: !Select
        - 0
        - !GetAZs
      CidrBlock: !Select
        - 0
        - !Cidr
          - !Ref CidrBlock
          - 8
          - 8
    TemplateURL: https://owen-public-production-bucket.s3.amazonaws.com/vpc-demo/vpc/subnet/subnet.template.yaml
```

You'll notice above that we are using `Type: AWS::CloudFormation::Stack` for our Subnet above. We're utilizing a Cloudformation code abstraction tool called Nested Stacks. This lets us define how we want our Subnet and associated resources configured once, and deploy it multiple times without repeating ourselves.

Now getting into the Subnet stack code:

Just like we said above, a Subnet lives in a single AZ, has a CidrBlock associated with it, and is attached to a VPC.

```yaml
Subnet:
  Type: AWS::EC2::Subnet
  Properties:
    AvailabilityZone: !Ref AZ
    CidrBlock: !Ref CidrBlock
    MapPublicIpOnLaunch: True
    VpcId: !Ref VPC
```

### Public Subnets

You'll also notice the parameter `MapPublicIpOnLaunch: True`. Setting this parameter to true, will give every resource deployed in this subnet a Public IP address. This will let it access the internet and be accessed from the internet. We can configure a Security Group to limit access to our public resources.

This subnet is commonly called a _Public Subnet_, because all resources in it receive their own Public IP. _Private Subnets_, on the other hand, are subnets in which resources do not have public IP addresses.

## Route Table Code

Again, we are using a Nested Stack to define our Route Table. The key parameters we are passing in:
  - VPC, a route table must be associated with a VPC.
  - IGW, we want a route to the internet, so we pass in our IGW.
  - Subnet, we will be attaching our Route Table to our subnet.

```yaml
RouteTable:
  Type: AWS::CloudFormation::Stack
  Properties:
    Parameters:
      VPC: !Ref VPC
      IGW: !Ref IGW
      Subnet: !Ref Subnet
    TemplateURL: https://owen-public-production-bucket.s3.amazonaws.com/vpc-demo/vpc/subnet/route-table/route-table.template.yaml
```

Provisioning a route table is very simple, we just need to provide a VPC to attach it to.

```yaml
RouteTable:
  Type: AWS::EC2::RouteTable
  Properties:
    VpcId: !Ref VPC
```

Next, we associate our Route Table with our subnet.

_Note: If we wanted, we could make one route table and associate it with every subnet we have._

```yaml
RouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    RouteTableId: !Ref RouteTable
    SubnetId: !Ref Subnet
```

Once we have associated our Route Table to our subnet, we add a route to the internet.

## Route Code

Routes have:
  - A Destination: Where you are trying to go.
  - A Target: How you get there.

```yaml
InternetRoute:
  Type: AWS::EC2::Route
  Properties:
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref IGW
    RouteTableId: !Ref RouteTable
```

Route tables let your computers ask for directions:

_How do I get to the internet (69.254.69.254)?_
_Take the IGW._

```bash
$ aws ec2 describe-route-tables
```

VPC automatically makes a route for you for intra-vpc routing, so you don't need to make routes to link your subnets together.

```json
{
    "DestinationCidrBlock": "10.0.0.0/16",
    "GatewayId": "local",
    "Origin": "CreateRouteTable",
    "State": "active"
}
```

## More to Come

  - Nat gateways
  - VPC Endpoints
  - VPC Peering
  - Private routing
  - VPC hosted Lambda Functions
  - Internal only Resources

## Deploy and Enjoy

[Have fun coding! Here are my links if you want to follow me on the internet!](https://owen.dev)
