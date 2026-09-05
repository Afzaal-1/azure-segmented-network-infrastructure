# Lessons Learned

This project helped me understand how Azure networking services work together in a segmented environment.

## Network segmentation

Using separate subnets for the web, API, and database tiers made the traffic flow much easier to understand.

Instead of allowing broad communication between every server, I had to decide exactly which tier needed access to the next one.

The final design allowed:

- Internet → Web on TCP 80
- Web → API on TCP 8080
- API → Database on TCP 3306

Direct Web → Database traffic was blocked.

## NSG rule priority matters

One of the most useful troubleshooting exercises was changing the priority of the API deny rule.

Even though the correct allow rule existed, traffic still failed when the deny rule was processed first.

This made it clear that NSG troubleshooting should include checking rule priority, not only the source, destination, protocol, and port.

## Routing can break a healthy peering connection

I also learned that connected VNet peering does not automatically mean traffic will reach the destination.

A bad user-defined route with the next hop set to `None` caused traffic to fail even though the peering itself was still configured.

Azure Network Watcher Next Hop was useful for confirming where Azure was trying to send the traffic.

## Peering settings need to be checked

During the peering test, I disabled virtual network access on the web-to-services peering.

The peering relationship still existed, but traffic from the web VNet to the services VNet stopped working.

This showed me that checking only whether a peering exists is not enough. The access settings also matter.

## Private DNS makes internal communication easier

Using the `cloudlab.internal` private DNS zone made the environment easier to work with.

Instead of relying only on private IP addresses, I could use names such as:

- `api01.cloudlab.internal`
- `db01.cloudlab.internal`

This also showed me why internal DNS is important in larger environments where IP addresses are harder to manage manually.

## Bastion and NAT Gateway solve different problems

At first, it was easy to think of both services as ways to give private VMs network access, but they solve different problems.

Azure Bastion was used for private administrative access to the VMs.

NAT Gateway was used for outbound Internet access from a private subnet so the database VM could reach package repositories.

This helped me understand the difference between inbound administration and outbound connectivity.

## Application problems are not always application problems

When `curl` or `nc` failed, the issue could have been caused by:

- NSG rules
- Route tables
- VNet peering
- DNS
- The service not listening on the expected port

Using Network Watcher together with command-line testing helped me narrow down the actual cause instead of guessing.

## Final takeaway

The biggest lesson from this project was that building the network was only part of the work.

The troubleshooting exercises were more valuable because they forced me to verify each layer of the connection and understand why traffic was allowed or blocked.

I now feel more comfortable working through Azure connectivity issues in a structured way instead of changing settings randomly.
