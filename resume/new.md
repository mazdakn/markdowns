# MAZDAK NASAB

[mazdak.nasab@gmail.com](mailto:mazdak.nasab@gmail.com) | [github.com/mazdakn](https://github.com/mazdakn) | [linkedin.com/in/mazdakn](https://linkedin.com/in/mazdakn/)

Staff Network & Systems Engineer with 15+ years of experience building secure, high-performance distributed systems, Linux kernel data planes, and cloud-native networking infrastructure. Core developer and maintainer of Project Calico, with deep expertise in eBPF/XDP, Linux networking internals, distributed routing, and declarative security policy.

## EXPERIENCE
**Member of Technical Staff** | Tailscale, Vancouver, Canada | 2026 – Present
* Engineering core networking functionality, userspace data-plane components, and secure mesh routing for Tailscale's global networking platform.

**Staff Software Engineer / Core Developer of Project Calico** | Tigera, Vancouver, Canada | 2021 – 2026
* Architected and maintained Project Calico, a widely adopted Kubernetes networking and security platform deployed across on-premises data centers and AWS, Azure, and GCP.
* Received the **Technical Pinnacle Award (2025)** and **Excellence Award (2025)** for top engineering contributions and domain leadership.

* **Policy Engine & Data Plane:**
  * Architected hierarchical network security policy enforcement and implemented support for Kubernetes `AdminNetworkPolicy` and `BaselineAdminNetworkPolicy`.
  * Implemented high-performance stateless firewalling in the Calico eBPF data plane using XDP programs for line-rate packet filtering.

* **Egress Traffic Engineering:**
  * Designed and built policy-based egress routing, enabling fine-grained traffic steering through dedicated egress gateways to satisfy enterprise compliance and security requirements.
  * Engineered Azure egress gateway integrations to govern and audit outbound workload traffic.
  * Implemented egress traffic prioritization using DiffServ/DSCP marking to integrate workload traffic seamlessly with underlying enterprise network fabrics.

* **Routing & Control Plane:**
  * Led the design and development of an in-cluster routing engine that programs cluster-wide routes without requiring BGP or external routing protocols, reducing operational complexity and improving scalability.
  * Led development of a high-performance userspace ARP/NDP responder, enabling workload and service reachability across complex L2/L3 topologies.
  * Extended BGP route reflection and advertisement capabilities with advanced multi-attribute route filtering.

* **Observability & Security:**
  * Built network observability pipelines correlating socket lifecycle and policy evaluation state to stream granular network flow logs.
  * Engineered workload runtime-security telemetry using eBPF to trace and analyze kernel syscalls and process execution events.

**Senior Site Reliability Engineer – Network Specialist** | Ericsson, Sweden | 2017 – 2021
* Designed and operated virtual network services (firewalls, L4/L7 load balancers, and proxies) for an IoT platform serving millions of connected devices.
* Hardened multi-tenant infrastructure using reverse proxies, multi-layer firewalling, and DoS mitigation systems.
* Engineered an in-house Linux-based microsegmentation firewall, saving over **$200,000** in third-party vendor licensing costs.

**Senior Software Engineer** | Enea Software, Sweden | 2015 – 2017
* Contributed to upstream OPNFV, porting core network virtualization components to ARM64 architectures.
* Certified Linux Foundation Trainer; instructed enterprise engineers in open-source virtualization technologies including KVM/QEMU, Linux namespaces, and container internals.

**System Developer** | Ericsson, Sweden | 2013 – 2015
* Re-architected a legacy TCP/HTTP proxy into a highly available, horizontally scalable Virtual Network Function (VNF).
* Automated provisioning and configuration workflows with Python and Ansible, reducing deployment times from hours to minutes.

## EDUCATION
**M.Sc. in Computer Science – Networks and Distributed Systems**
Chalmers University of Technology, Sweden | 2013

## SKILLS
* **Languages:** Go, C, Python, Lua, Bash
* **Linux Kernel & Networking:** eBPF, XDP, Netfilter (iptables/nftables), TC (Traffic Control), Socket API, BGP, ARP/NDP, WireGuard, VXLAN, Geneve
* **Network Security:** Firewalls, network policy, microsegmentation, egress control, DoS mitigation, security telemetry
* **Distributed Systems & Cloud:** Kubernetes, Docker, Linux, AWS, Azure, GCP, Ansible, Git
