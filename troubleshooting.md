# Troubleshooting Notes

This file documents the network issues I created and fixed during the Azure Segmented Network Infrastructure project.

The goal was to practice finding the cause of connectivity problems instead of only building resources when everything was already working.

## 1. NSG Priority Issue

### What I changed

The API subnet had an allow rule for Web → API traffic on TCP 8080 and a broader deny rule for the same port.

To create a failure, I changed the deny rule so it had a higher priority than the allow rule.

In Azure NSGs, a lower priority number is processed first.

### What happened

Web → API traffic stopped working.

A request that worked before:

```bash
curl http://10.20.1.4:8080
```

started timing out.

### How I diagnosed it

I used Azure Network Watcher IP Flow Verify.

The result showed that the traffic was being denied by:

`Deny-Other-API-Traffic`

This confirmed that the NSG rule order was the cause of the problem.

### Fix

I restored the rule order so the Web → API allow rule was processed before the broader deny rule.

The final rule order was:

- Allow-Web-To-API — priority 100
- Deny-Other-API-Traffic — priority 200

After the change, Web → API connectivity worked again.

### What I learned

NSG rules are not only about source, destination, and port. Priority is also important because Azure processes the matching rule with the lower priority number first.

---

## 2. Route Table / UDR Issue

### What I changed

I created a route affecting traffic from the web subnet to the services network.

The route used:

- Destination: `10.20.0.0/16`
- Next hop: `None`

This intentionally caused traffic to the services network to be dropped.

### What happened

The web server could no longer reach the API server in the peered VNet.

The same API request that previously worked started failing.

### How I diagnosed it

I used Azure Network Watcher Next Hop to check traffic from:

- Source: `vm-web01` / `10.10.1.4`
- Destination: `10.20.1.4`

Network Watcher showed the next hop as:

`None`

This matched the route I created and confirmed that routing was the cause of the failure.

### Fix

I removed the bad route so the normal Azure system route for VNet peering could be used again.

After the route was removed, Web → API connectivity was restored.

### What I learned

A peering connection can be healthy while traffic still fails because of routing.

Checking the actual next hop helped separate a routing problem from an NSG or peering problem.

---

## 3. VNet Peering Issue

### What I changed

I temporarily disabled virtual network access on the web-to-services peering from `vnet-web` to `vnet-services`.

### What happened

Traffic from the web VNet to the services VNet stopped working.

Web → API connectivity failed even though the NSG rules and VM services were still configured correctly.

### How I diagnosed it

I reviewed the peering configuration and found that virtual network access was disabled.

This explained why traffic could no longer pass between the two VNets.

### Fix

I re-enabled virtual network access on the web-to-services peering.

After restoring the setting, Web → API communication worked again.

### What I learned

VNet peering can appear to exist while communication is still blocked by the peering settings.

When troubleshooting cross-VNet traffic, I need to check both the peering status and the access settings.

---

## Final Validation

After fixing the test failures, I used Azure Network Watcher Connection Troubleshoot to validate Web → API connectivity.

The final result showed:

- Destination: Reachable
- Outbound NSG: Allow
- Next hop type: VirtualNetworkPeering
- Route source: System Route

I also verified the expected application paths:

### Web → API

```bash
curl http://10.20.1.4:8080
```

Result: successful

### Web → API using Private DNS

```bash
curl http://api01.cloudlab.internal:8080
```

Result: successful

### API → Database

```bash
nc -zv -w 5 db01.cloudlab.internal 3306
```

Result: successful

### Web → Database

Direct connectivity from the web tier to TCP 3306 was tested.

Result: blocked as designed

## Main Troubleshooting Tools Used

- Azure Network Watcher IP Flow Verify
- Azure Network Watcher Next Hop
- Azure Network Watcher Connection Troubleshoot
- `curl`
- `nc`
- Azure NSG rule review
- Azure route table review
- Azure VNet peering settings

## Summary

The most useful part of this project was seeing how similar connectivity failures can come from different parts of the network.

An application timeout by itself did not show whether the cause was an NSG, route table, peering setting, or the service on the VM.

Using Azure Network Watcher together with command-line testing made it much easier to narrow down the problem and verify the fix.
