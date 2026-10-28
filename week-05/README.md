# Week 5: VPC Deep Dive

## What I built
A custom VPC (`training-vpc`, 10.0.0.0/16) across two Availability Zones, with
two public and two private subnets, an Internet Gateway, and one NAT Gateway.
I launched a bastion host in a public subnet and a second instance in a private
subnet to practice the bastion pattern.

## Traffic path: laptop to private instance
1. My laptop starts an SSH connection to the **bastion's public IP** over the
   internet.
2. The traffic enters the VPC through the **Internet Gateway**. The public
   subnet's route table sends `0.0.0.0/0` to the Internet Gateway, which is
   what makes that subnet "public".
3. The **bastion's security group** allows SSH (port 22) only from my IP, so the
   connection is accepted.
4. From the bastion, I SSH to the private instance's **private IP**. This
   traffic never leaves the VPC. It uses the built-in `local` route.
5. The **private instance's security group** allows SSH only from the
   bastion's security group, so it accepts the connection from the bastion and
   rejects everything else.
6. I used SSH agent forwarding (`ssh -A`) so my private key stays on my laptop
   and is never copied to the bastion.

## Traffic path: private instance to the internet
The private instance has no public IP. Its subnet's route table sends
`0.0.0.0/0` to the **NAT Gateway** (in a public subnet), which forwards the
request through the Internet Gateway using its Elastic IP. Replies come back
the same way, but the internet can't start a connection to the private instance.

## Key takeaways
- A subnet is "public" because of its route table, not its name.
- Security groups can reference other security groups, so access follows the
  role of the instance instead of an IP address that changes.
- NAT Gateways cost about $0.045/hr, so I deleted it at the end of the week.

## Cleanup
Terminated both instances, deleted the NAT Gateway, released the Elastic IP,
and deleted the VPC.