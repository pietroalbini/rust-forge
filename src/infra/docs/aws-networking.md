# AWS Networking

AWS uses VPCs (Virtual Private Clouds) to define the network hosting the
instances and other resources deployed in our account. This documentation page
explains our VPC setup, and how to tweak it. The setup is managed with
Terraform, and its source can be found in simpleinfra's
[`terraform/vpc.tf`][vpc.tf] and [`terraform/modules/vpc/`][modules/vpc].

[vpc.tf]: https://github.com/rust-lang/simpleinfra/blob/master/terraform/vpc.tf
[modules/vpc]: https://github.com/rust-lang/simpleinfra/tree/master/terraform/modules/vpc/

## Production VPC

The production VPC, called `rust-prod`, is the main VPC for the project and
it's located inside the `us-west-1` (Northern California) region. All the
workloads should be deployed inside it, unless there is an explicit reason not
to.

* IPv4 CIDR: `10.0.0.0/16`
* IPv6 CIDR: `2600:1f1c:f21:3700::/56`

### Public subnets

Public subnets are meant to host proxies and gateways to resources in our
private subnets, and resources inside them can be reached from the outside. All
their traffic received or sent outside the VPC is routed through an Internet
Gateway.

Workloads should not be deployed in a public subnet unless there is an explicit
reason to do that (such as the [bastion server](bastion.md)). Alternatives to
deploying something in a public subnet are:

* If your instance needs to serve HTTP or HTTPS content consider putting it
  inside a private subnet and using the load balancer to forward requests to
  it.
* If you need to access the instance remotely consider putting it inside a
  private subnet and using the bastion to log into it.

Everything inside a public subnet must have a security group restricting
inbound access as much as possible. Keep in mind everything on the Internet or
inside an untrusted subnet can communicate with instances inside the public
subnet unless you explicitly prevent that with security groups.

### Private subnets

Private subnets are meant to host most of our resources, and while resources
inside them can initiate connections to the whole Internet, they can't be
reached from the outside.

The limitation is implemented inside the subnet's route table. IPv4 traffic
received or sent outside the VPC is not routed through an Internet Gateway, but
through a NAT Gateway located inside a public subnet. IPv6 traffic is routed
through an egress-only Internet Gateway that prevents inbound connections:
there are no private IPv6 addresses on AWS, so doing NAT is not necessary.

There are currently two NAT Gateways in our VPC, each of them deployed in a
different Availability Zone to prevent cutting off all network access if an AZ
goes down. To reduce NAT costs, VPC endpoints for S3 and DynamoDB are also
configured inside the route table: requests to those two services in the
`us-west-1` region will not have any extra NAT charge.

The effect of this routing is twofold:

* All the outgoing IPv4 traffic from each Availability Zone will come from the
  same IP address.
* There is no way to reach a resource inside a private subnet, even if inbound
  connections are explicitly allowed by the resource's security group.

### Untrusted subnets

Untrusted subnets are intended to be used for services that run untrusted code,
or that are managed by other people we don't fully trust. Their configuration
is similar to private subnets: all their IPv4 traffic goes through NAT
Gateways and they're not reachable from the outside.

The difference between private and untrusted subnets is that any communication
from an host inside an untrusted subnet to other hosts inside untrusted or
private subnets is prevented at the network level, overriding any security
group allowing that kind of access.

This is accomplished by a custom Network ACL, that allows connections to
`0.0.0.0/0` and `::/0` while denying connections to the private and untrusted
subnets' IP ranges.

### Adding capacity

Each subnet inside the VPC is a `/24`, so it has 256 IPv4 addresses available.
If a subnet runs out of IP addresses and you need to spawn an instance, you can
add another subnet of the same kind inside the VPC and spawn the workload
there.

To add new subnets, change the [`terraform/vpc.tf`][vpc.tf] file of the
simpleinfra repository to include the new ones you want to add. Inside the
module defintion, there are maps for all the kinds of subnets we support: the
keys represent the third triplet of the IPv4 (i.e. "12" in `10.11.12.13`),
while values are the name of the Availability Zone in which that subnet is
deployed.

For example, if you want to add a private subnet with an IPv4 range of `10.0.42.0/24`
living in the `usw1-az1` Availability Zone you can add this to the map in the
module definition:

```
  private_subnets = {
    # ...
    42 = "usw1-az1",
  }
```

> **Note:** some of the services we use (such as ECS with Fargate) are trying
> to balance their workloads across Availability Zones. When you need to add
> capacity always try to add **two** subnets of that kind instead of just one,
> adding one in each AZ. This way those services will be able to keep the
> balance without running out of capacity themselves.

## Legacy VPC

In the past our infrastructure used another VPC, `rust-legacy`. While some
resources are still deployed inside it, we want to eventually migrate all of
them inside the `rust-prod` VPC. No new resources should be spawned there.

* IPv4 CIDR: `172.30.0.0/16`
* IPv6 CIDR: disabled
