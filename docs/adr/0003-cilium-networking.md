# ADR-0003: Cilium networking

## Status
Accepted

## Context
`eks-prod` needs a CNI. The platform's security requirements call for default-deny network
policy, explicit service-to-service policy, and network flow visibility — not just "pods
can reach each other," which the default AWS VPC CNI provides with no policy layer of its
own.

## Decision
Cilium as the primary CNI, used intentionally: `CiliumNetworkPolicy` for default-deny plus
explicit allow-lists, Cilium Gateway API for the one external ingress point (the Shopify
webhook NLB), Hubble for flow visibility, and full kube-proxy replacement (`vxlan` overlay
mode with cluster-pool IPAM — simpler than native ENI mode, and avoids ENI IP exhaustion on
small node counts).

## Alternatives considered
- **Default AWS VPC CNI + a separate ingress controller** (e.g. AWS Load Balancer
  Controller + Calico for policy): more moving parts, more IAM permissions, and still needs
  a policy engine bolted on separately to get what Cilium provides natively.
- **A managed API Gateway** (API Gateway + NLB): adds AWS-proprietary surface for a single
  webhook endpoint that doesn't need the extra features.

## Consequences
One NLB total, provisioned through Cilium's Gateway API implementation rather than a
separate ingress controller — keeps the load-balancer count (and its ongoing cost) to
exactly what's needed. Kube-proxy replacement means iptables-based service routing is
gone entirely, which is also a meaningful operational simplification once `kubectl` and
`hubble` are the tools used to reason about traffic instead of an opaque iptables ruleset.
