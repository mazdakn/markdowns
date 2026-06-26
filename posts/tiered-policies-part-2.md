# Operating Tiered Policy at Scale: Calico Tiers, Shadow Rules, and Team Ownership

*Part 2 of a two-part series on the hierarchy of trust. [Part 1](tiered-policies-part-1.md) covered why flat network policy breaks down at scale and how the native Kubernetes `ClusterNetworkPolicy` API answers it with a fixed three-layer hierarchy. This post extends that model with Calico, shows how the two interoperate, and digs into what running tiers in production actually costs.*

In [Part 1](tiered-policies-part-1.md) we established why a flat network policy plane buckles at enterprise scale, and how a tiered model fixes it with four primitives: cluster-wide scope, RBAC-gated tiers, deterministic top-down evaluation, and the Pass action that lets a high-priority tier delegate the final verdict downward. The native Kubernetes `ClusterNetworkPolicy` API delivers all four in a fixed `Admin → NetworkPolicy → Baseline` stack. That stack is a great default—but "exactly three layers" is also its ceiling. Here we pick up where it leaves off.

## Extending the Model: Calico Policy Tiers

While the native Kubernetes APIs introduce a great three-layer model, enterprise environments often require finer granularity. Calico expands on this concept by offering Policy Tiers—allowing you to design an arbitrary number of custom evaluation layers. In Calico, every network policy lives within a designated tier, and traffic is evaluated through those tiers sequentially, in ascending order of each tier's assigned order value (lowest integer first). Within a hierarchy of trust, a typical enterprise stack maps directly to team responsibilities: Security tiers → Platform tiers → Application tiers.

Calico offers two policy resource types that live inside tiers: namespace-scoped `NetworkPolicy` for team- and app-local rules, and cluster-scoped `GlobalNetworkPolicy`, which is non-namespaced and selects endpoints across the entire cluster—pods, VMs, even host interfaces. GlobalNetworkPolicy is how Calico satisfies the cluster-wide scope requirement: a security or platform team applies one resource that covers all current and future namespaces, the direct analog to a ClusterNetworkPolicy in the Admin or Baseline tier. (Note the name collision: Calico's own `NetworkPolicy` in the `projectcalico.org/v3` API group is a different resource from the upstream Kubernetes `NetworkPolicy`.)

Each tier holds its own ordered list of policies and ends in a configurable default action; evaluation flows top-down until a rule (or a tier default) returns a terminal Allow/Deny, while a Pass cascades to the next tier:

```
┌──────────────────────────────────────────────────────────────┐
│ Tier (order: 100)            RBAC owner: e.g. Security       │
│   ├─ policy  (order: 10) ─┐                                  │
│   ├─ policy  (order: 20)  │  rules evaluated by policy order │
│   └─ ...                  ▼  → Allow | Deny | Pass           │
│   end-of-tier default: Pass | Deny  (you choose, per tier)   │
└───────────────────────────────┬──────────────────────────────┘
                                │ (If Pass or no match)
                                ▼
┌──────────────────────────────────────────────────────────────┐
│ Tier (order: 200)            RBAC owner: e.g. Platform       │
│   └─ policies … → Allow | Deny | Pass                        │
│   end-of-tier default: Pass | Deny                           │
└───────────────────────────────┬──────────────────────────────┘
                                │ (If Pass or no match)
                                ▼
                              ⋮   (arbitrary additional tiers)
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│ Tier: default (order: highest)   RBAC owner: e.g. Developers │
│   └─ policies … → Allow | Deny | Pass                        │
│   end-of-tier default: Deny  (fail-closed safety net)        │
└──────────────────────────────────────────────────────────────┘
```

The shape is the point: where the native model hands you exactly three fixed layers, Calico's tiering is a generic, extensible primitive. The number, names, and ordering are yours, and every tier carries its own end-of-tier default action and RBAC scope—so the Security → Platform → Application tiers above are just one convention, not a hard-coded ceiling.

### The Nuance of Default Actions: Native vs. Calico

When designing your policy architecture, it is vital to account for how a tier behaves when a packet fails to match any explicitly defined rules inside it. Under the native Kubernetes ClusterNetworkPolicy specification, the default behavior at the end of a tier is implicitly Pass, gracefully moving the packet along to the next layer of evaluation. Calico defaults the other way—to an implicit Deny—but with an important difference: that implicit Deny only applies when the workload is selected by at least one policy in the tier yet matches none of that tier's rules. If *no* policy in the tier selects the workload at all, the tier is simply skipped and the packet passes through to the next tier; it is not dropped. Calico also gives administrators the flexibility to override this end-of-tier behavior on a per-tier basis, setting the default action to either Deny or Pass depending on whether you want a strict isolation barrier or an open transit corridor.

### Bridging the Standards Gap

A significant benefit of Calico architecture is its native compatibility with Kubernetes standards. Calico deployments based on v3.32+ releases automatically ingest cluster-wide ClusterNetworkPolicy resources, mapping their implicit admin and baseline tiers into native Calico tiers. For example, Admin-tier ClusterNetworkPolicy resources land in an auto-created tier, named kube-admin (order 1,000, default action Pass), and Baseline-tier resources in another auto-created tier, named kube-baseline (order 10,000,000, default action Pass), so they slot into Calico's evaluation order without any manual wiring. This ensures that you can design an open-source standard architecture while still taking advantage of Calico's optimized data plane enforcement.

### Per-Tier RBAC: Delegating Ownership Without Sharing Keys

Because Calico models tiers as first-class Kubernetes resources, it lets you set RBAC access on a *per-tier* basis—a level of granularity the native API's all-or-nothing CRD access can't express. The result is the separation of concerns this model promises: the security team can fully own the security tier while developers are confined to the default tier, each team isolated from the others' configurations.

## The Complexity Trap: The Real-World Challenges of Tiered Policies

As powerful as policy tiers are for establishing a clear hierarchy of trust, they introduce a distinct operational paradigm shift. Moving from a flat rule plane to a multi-layered execution environment solves the visibility and guardrail problems, but it introduces a new operational cost: **cognitive complexity**.

If you are planning to roll out tiers across your production clusters, you need to be prepared to tackle two main challenges:

1. **The Troubleshooting Challenge:** While the Pass action is essential for delegating control, it creates an invisible tracing problem. If a packet enters the cluster and encounters three separate policies in the Security tier, two in the Platform tier, and five in the developer's Application tier before finding a match, maintaining a mental model of that packet's lifecycle becomes very hard. Because each Pass action pushes evaluation to the next subsequent layer without finalizing a verdict, debugging a dropped packet requires tracing state across multiple files, distinct Kubernetes resources, and varying organizational personas. You are no longer just looking at a YAML file; you are evaluating an execution stack.

2. **The Anatomy of a Shadow Rule:** A subtler challenge in a tiered environment is policy shadowing—specifically, when a rule in a higher tier neutralizes or masks a valid intent in a lower tier without throwing any errors. This generally happens when a broad rule in a high-precedence tier like the Security tier impacts application traffic. As an example, the security team might deploy a global compliance policy intended to simply audit or log a specific type of traffic. However, if they forget to terminate that policy with an explicit Pass action, they will unintentionally capture that pod's traffic. The packet will be cleanly dropped at the end of the Security tier (whose default action is Deny), completely starving out the developer's downstream application rules without throwing an explicit syntax error during deployment.

To mitigate these challenges, it is important to consider best practices in policy model design.

## Practical Architecture: Designing Your Tiers

To build a stable cluster defense layout, you shouldn't create a dozen chaotic tiers. The best practice is to divide ownership into distinct personas, with each persona owning its own tiers:

| Persona | Core Responsibility | Example Use Case |
| :---- | :---- | :---- |
| **Security Engineers** | Global threat mitigation & absolute boundaries. | Block Log4Shell callback egress; quarantine compromised namespaces; block cloud metadata APIs. |
| **Platform Engineers** | Infrastructure logging, metrics, and mesh stability. | Ensure Prometheus can scrape endpoints cluster-wide; allow standard CoreDNS egress. |
| **Developers** | Microservice-to-microservice functional connectivity. | Allow frontend pods to communicate with backend pods on port 8080. |

This maps cleanly onto either model: in the native API, Security and Platform share the Admin and Baseline tiers (strict Deny guardrails and fail-closed defaults respectively) while Developers own namespace-scoped NetworkPolicy in the middle. In Calico, each persona owns its own RBAC-gated tier—Security low-order with an end-of-tier Deny, Platform in the middle typically defaulting to Pass, and Developers confined to the high-order default tier that closes fail-closed.

## Summary

Transitioning from flat network policies to tiered architectures is the cloud-native equivalent of moving from a chaotic, single-file legacy firewall script to a clean, structured enterprise firewall zone layout. Standard Kubernetes NetworkPolicy alone can't get you there—it's namespace-jailed, allow-only, and strictly additive, which leaves a hard persona gap between the teams who set guardrails and the teams who ship services.

Tiered policies close that gap with four primitives: cluster-wide scope, strict RBAC control, deterministic top-down evaluation, and the Pass action that lets a high-priority tier delegate the final verdict downward. The native Kubernetes ClusterNetworkPolicy API bakes this into a fixed Admin → NetworkPolicy → Baseline stack, while Calico generalizes it into an arbitrary, RBAC-gated ordering of tiers with per-tier default actions. Both demand the same operational discipline—watch for Pass-tracing blind spots and shadow rules, enforce per-tier ownership, and keep the stack lean.

The payoff is that you no longer have to choose between absolute security and developer velocity. By separating cluster security guardrails from application development agility, tiered policies deliver both: security teams can trust that their boundaries cannot be bypassed, while application developers retain full control over their microservice configurations without bureaucracy.
