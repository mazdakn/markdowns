# Tiered Network Policy: Scaling Kubernetes Security Beyond Flat Rules

As Kubernetes clusters scale from a few development sandboxes to massive, multi-tenant production environments, platform teams often find themselves facing a configuration management crisis—a small number of microservices suddenly demand hundreds of individual Kubernetes NetworkPolicy objects. Managing them becomes operationally expensive, auditing them is difficult, and a single developer misconfiguration can easily drop critical production traffic or open a massive security hole.

To scale cluster security without slowing down engineering velocity, we must abandon the flat, uncoordinated rule planes of the past. The solution lies in establishing a clear, multi-layered framework: a hierarchy of trust powered by tiered network policies.

## The Core Problem with Standard Kubernetes NetworkPolicy

Standard Kubernetes NetworkPolicy resources are genuinely useful for basic application microsegmentation, but they have major architectural and organizational bottlenecks when scaled across an enterprise:

1. **Namespace-Scoped by Design:** Standard network policies are inherently scoped to a namespace. If your security team mandates a cluster-wide rule—such as blocking all internal pods from querying the cloud provider's metadata API (169.254.169.254)—you have to copy-paste that policy into *every single namespace*. If a developer creates a new namespace, that guardrail doesn't exist until someone manually applies it.
2. **Organizational Friction:** Because anyone with namespace access can manipulate these policies, it creates a persona gap within organizations. Platform & Security teams need to enforce global, un-overrideable guardrails (e.g. "Isolate the payments namespace from everything else"). DevOps teams need the freedom to write granular, service-to-service rules for their applications without opening infrastructure support tickets.
3. **No Rules Hierarchy:** Kubernetes network policies are strictly additive. There are no weights, priorities, or order sequences. An application developer can accidentally (or intentionally) write a loose policy that bypasses the security team's intended restrictions, undermining any baseline trust.
4. **The "Allow-Only" Restriction:** Standard policies cannot explicitly Deny traffic. They operate solely on an *allow-list* model. Isolation is implicit: if a pod is selected by a policy, any traffic not explicitly allow-listed is dropped. This makes it impossible to write a simple, top-level rule that says, *"Block traffic from Namespace X to Namespace Y, no matter what."*

## What a Scalable Solution Requires

To solve these scaling pain points, we have to move away from a flat network architecture and adopt a Tiered Policy Model. A scalable solution requires four core capabilities:

1. **Global, Cluster-Wide Scope:** To stop copy-pasting rules, administrators need a policy type that natively operates at the cluster level rather than the namespace level. This allows a single manifest to apply to all current and future namespaces automatically, eliminating the risk of "configuration drift" and ensuring day-one protection for new workloads.
2. **Separation of Concerns (RBAC-Gated Tiers):** Security, platform, and application teams need their own distinct logical "zones" or tiers to deploy rules. These tiers must be strictly gated by Role-Based Access Control (RBAC) so a developer modifying their application namespace cannot alter or override a higher-priority platform or security tier.
3. **Deterministic, Top-Down Evaluation:** The firewall engine must evaluate these tiers sequentially. Traffic must pass through the highest-priority tier (e.g., Security) before it ever reaches a lower tier (e.g., Application).
4. **Explicit Deny and Pass Actions:** Standard policies are allow-only, so they can never express a hard "block this, period." A tiered model needs explicit actions: a Deny that states a prohibition outright, and—crucially—a third option, Pass, that lets one tier defer the decision to the next rather than ending it (covered in detail below).

### Why the Pass Action Matters

The key enabler of tiered policies is the Pass action. Think of Pass as a delegated hand-off. When a packet matches a rule with a Pass action in a high-priority tier, the engine skips the remaining lower-precedence rules in that tier and continues evaluation in the next tier down the hierarchy. This allows security administrators to say: "This traffic is safe by our standards, but we aren't explicitly endorsing it. We are passing the final decision down to the platform or development teams to handle at their layer." Without a Pass action, tiered policies become brittle, forcing admins to explicitly track and Accept every single microservice connection at the highest level, which would defeat the purpose of developer agility.

## The Kubernetes Native Answer: ClusterNetworkPolicy

Recognizing these scalability constraints, the SIG-Network Network Policy API group developed a native, multi-layered solution: **ClusterNetworkPolicy**. The API delivers exactly the four capabilities outlined above, with a few concrete specifics worth calling out:

* **A Native Three-Layer Hierarchy:** It introduces distinct, sequentially evaluated resource tiers—ClusterNetworkPolicy (Admin tier) at the top for absolute guardrails, standard NetworkPolicy in the middle for developer agility, and ClusterNetworkPolicy (Baseline tier) at the bottom as a cluster-wide fallback safety net. Unlike namespace-jailed standard policies, the Admin and Baseline tiers apply across the entire cluster.
* **Separation of Concerns:** Because ClusterNetworkPolicy is delivered as a new Custom Resource Definition (CRD) rather than a tweak to the existing NetworkPolicy type, standard Kubernetes RBAC governs who can interact with it. This ensures that only Security/Platform teams access ClusterNetworkPolicy resources, while DevOps teams work only with namespaced network policies.
* **Numeric Precedence:** Policies feature explicit integer priorities. A policy with a lower integer value (e.g., 10) takes precedence over a policy with a higher value (e.g., 100), allowing for deterministic evaluation.
* **Explicit Actions:** Rules are no longer purely additive—you can now design rules with explicit Accept (formerly named Allow), Deny, and Pass actions.

This API completely shifts how cluster administrators manage traffic by introducing a native, three-tiered evaluation hierarchy:

```
┌────────────────────────────────────────────────────────┐
│ 1. ClusterNetworkPolicy (Admin tier)                   │
│    Scope: Cluster-wide                                 │  ◄── Strict Guardrails (Security/Platform)
│    Actions: Accept, Deny, Pass                         │
└───────────────────────────┬────────────────────────────┘
                            │ (If Pass or no match)
                            ▼
┌────────────────────────────────────────────────────────┐
│ 2. Standard NetworkPolicy                              │
│    Scope: Namespace                                    │  ◄── Application Logic (Developers)
│    Actions: Allow-only (implicit deny)                 │
└───────────────────────────┬────────────────────────────┘
                            │ (If no match)
                            ▼
┌────────────────────────────────────────────────────────┐
│ 3. ClusterNetworkPolicy (Baseline tier)                │
│    Scope: Cluster-wide                                 │  ◄── Default Fallbacks (Security/Platform)
│    Actions: Accept, Deny, Pass                         │
└────────────────────────────────────────────────────────┘
```

**The Top Layer: ClusterNetworkPolicy (Admin tier)**: This is the high-priority tier controlled by cluster and security administrators. Rules here are evaluated first, and two of its three actions are *terminal*: an Accept or a Deny is a final verdict that bypasses the developer's NetworkPolicy layer entirely. A Deny here cannot be overridden by any developer manifest—but the same is true of Accept: if an admin explicitly accepts traffic, it is permitted regardless of what a developer policy would have decided. This is the crucial difference from a standard NetworkPolicy allow, which is *additive*—an Admin-tier Accept is an override, not a contribution. Only the third action, Pass, is non-terminal: it declines to decide and hands evaluation down to the next tier.

**The Middle Layer: Standard NetworkPolicy**: This is the traditional application-developer tier. It only kicks in if traffic wasn't explicitly allowed or denied by the ClusterNetworkPolicy in the Admin tier above it. This keeps developers agile, letting them connect their microservices without needing admin intervention. One subtlety to keep in mind: standard NetworkPolicy carries an *implicit deny* for any pod it selects. So traffic only falls through to the Baseline tier when no NetworkPolicy selects the workload at all—a pod that *is* selected but matches none of its allow rules is already dropped here, and never reaches the Baseline tier below.

**The Bottom Layer: ClusterNetworkPolicy (Baseline tier)**: This is the cluster-scoped Baseline tier, meant for default fallbacks. It acts as the safety net after developer policies are checked. For example, if a developer forgets to secure their pod, this policy can enforce a default cluster-wide posture like "if no developer policy matches this traffic, deny all intra-cluster traffic by default."

Combined, these features provide a native, multi-level strategy for scaling enterprise cluster security far beyond the limitations of a flat configuration.

