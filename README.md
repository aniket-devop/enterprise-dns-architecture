<div align="center">

# Enterprise DNS Architecture on Azure

**Public DNS, Azure Private DNS and Azure Virtual Desktop, explained with one working design**

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![Azure Private DNS](https://img.shields.io/badge/Azure_Private_DNS-0078D4?style=flat)
![Azure Virtual Desktop](https://img.shields.io/badge/Azure_Virtual_Desktop-0078D4?style=flat)
![nginx](https://img.shields.io/badge/nginx-009639?style=flat&logo=nginx&logoColor=white)
![Type](https://img.shields.io/badge/Type-Architecture_Documentation-blue?style=flat)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

</div>

---

Almost every connection starts with a DNS lookup, and DNS problems are hard to debug because the lookup happens out of sight. This repository documents a design where DNS decides how two groups of users reach their applications: internet users reach a public dashboard, and internal users reach a private Benefits Portal through Azure Virtual Desktop.

> **Scope:** This repository documents an architecture design based on a diagram. It contains no deployment evidence, test output, monitoring data or production claims. Statements are labelled by origin:
>
> - **Diagram**: shown in the architecture diagram.
> - **Assumption**: not shown in the diagram, assumed to complete the documentation.
> - **Recommended**: a suggested implementation, not a description of what exists.

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Architecture Components](#3-architecture-components)
4. [Public DNS Resolution](#4-public-dns-resolution)
5. [Private DNS Resolution](#5-private-dns-resolution)
6. [Recursive DNS Resolution](#6-recursive-dns-resolution)
7. [Public DNS vs Private DNS](#7-public-dns-vs-private-dns)
8. [Azure Network Architecture](#8-azure-network-architecture)
9. [Benefits Portal Access Flow](#9-benefits-portal-access-flow)
10. [DNS Caching and Troubleshooting](#10-dns-caching-and-troubleshooting)
11. [Security Considerations](#11-security-considerations)
12. [Technologies Used](#12-technologies-used)
13. [Technical Learnings](#13-technical-learnings)
14. [Future Improvements](#14-future-improvements)
15. [Assumptions and Limitations](#15-assumptions-and-limitations)
16. [Repository Structure](#16-repository-structure)
17. [Author and Acknowledgements](#17-author-and-acknowledgements)

---

## 1. Project Overview

| Audience | Application | Name | DNS type |
|---|---|---|---|
| Internet users | Public dashboard | `dashboard.veltrixrockets.space` | Public DNS zone |
| Internal users, through Azure Virtual Desktop | Benefits Portal (nginx) | `portal.veltrixrocket.internal` | Azure Private DNS zone |

The design has three parts:

1. **Public DNS.** The domain `veltrixrockets.space` is delegated to name servers that host a public zone. The zone maps hostnames to public IP addresses.
2. **Private DNS.** An Azure Private DNS zone, `veltrixrocket.internal`, resolves `portal.veltrixrocket.internal` to `10.124.0.4` inside the virtual network `10.124.0.0/16`. The address is private and never published on the internet.
3. **Recursive resolution.** A step-by-step walkthrough of how a lookup moves from the client through the hosts file, local resolver, recursive DNS server, root, TLD and authoritative servers.

**What this project covers**

- How a domain is delegated and published
- How internal names are kept off the public internet
- How the public and private namespaces differ
- How to troubleshoot resolution on Linux and Windows

> **Note on spelling:** the public domain is `veltrixrockets.space` (plural) and the private zone is `veltrixrocket.internal` (singular). Both are documented exactly as they appear in the diagram.

## 2. Architecture Diagram

![DNS-Based Application Access Architecture](docs/images/architecture-diagram.png)

| Area | Content |
|---|---|
| Top left | Domain purchase, name servers and the public DNS zone with three records |
| Middle left | Internet users reaching `dashboard.veltrixrockets.space` at `54.56.21.45` |
| Bottom left | Azure Virtual Desktop environment in VNet `10.124.0.0/16`: Private DNS zone, Workstations subnet, Benefits Portal subnet |
| Bottom right | Recursive resolution: hosts file, local DNS, DNS client, cache, root, TLD and authoritative servers |

The flowchart below restates the diagram as text. It adds nothing that is not in the diagram.

```mermaid
flowchart TB
    subgraph PUBLIC["Public side"]
        REG["Domain purchase<br/>veltrixrockets.space"] --> NS["Name servers"]
        NS --> PZ["Public DNS zone"]
        PZ --> R1["veltrixrockets.space<br/>45.56.54.56"]
        PZ --> R2["dhondhu.veltrixrockets.space<br/>45.56.54.86"]
        PZ --> R3["dashboard.veltrixrockets.space<br/>54.56.21.45"]
        USERS["Internet users"] -->|"dashboard.veltrixrockets.space"| VM["Dashboard VM<br/>54.56.21.45"]
    end

    subgraph AZURE["Azure Virtual Desktop environment - VNet 10.124.0.0/16"]
        PDNS["Private DNS zone veltrixrocket.internal<br/>portal = 10.124.0.4"]
        subgraph WS["Workstations subnet"]
            W1["WORKSTATION1"]
            W2["WORKSTATION2"]
            W3["WORKSTATION3"]
        end
        subgraph BP["Benefits Portal subnet"]
            NGINX["nginx Benefits Portal<br/>10.124.0.4"]
        end
        W1 --> PDNS
        W2 --> PDNS
        W3 --> PDNS
        PDNS -.->|"resolves to"| NGINX
    end

    LAPTOPS["End users with laptops"] --> W1
    LAPTOPS --> W2
    LAPTOPS --> W3
```

## 3. Architecture Components

### 3.1 Domain and delegation

The diagram shows the sequence *Domain Purchase → Name Servers → Public DNS Zone*. The registrar records the domain's name servers (NS records) with the `.space` registry. When a resolver asks the `.space` TLD servers about `veltrixrockets.space`, it gets a referral to those name servers, which then answer from the public zone.

The DNS provider hosting the zone is not named in the diagram (see [section 15](#15-assumptions-and-limitations)).

### 3.2 Public DNS zone

| Record name | Value | Notes |
|---|---|---|
| `veltrixrockets.space` | `45.56.54.56` | Zone apex |
| `dhondhu.veltrixrockets.space` | `45.56.54.86` | The diagram does not show what this host serves |
| `dashboard.veltrixrockets.space` | `54.56.21.45` | The VM that internet users reach |

**Assumption:** the diagram does not label the record type. Each value is an IPv4 address, so the records are documented as A records. TTL values are not shown. The zone also holds NS and SOA records at the apex, which the diagram does not list.

### 3.3 Public application

Internet users connect to `dashboard.veltrixrockets.space`, which resolves to a VM at `54.56.21.45`. The operating system, web server, TLS setup and hosting platform are not shown.

### 3.4 Azure virtual network

The virtual network uses the address space `10.124.0.0/16` and contains two subnets: Workstations and Benefits Portal. Subnet prefixes are not shown.

### 3.5 Azure Private DNS zone

A Private DNS zone answers queries only for virtual networks linked to it. This zone holds one record in the diagram:

| Record | Value |
|---|---|
| `portal.veltrixrocket.internal` | `10.124.0.4` |

The `.internal` domain is reserved for private-use applications and is not meant for public registration or delegation. Use it only inside controlled private DNS environments.

**Assumption:** the `portal` record was created manually as an A record. Azure auto-registration creates records from VM names, so it would not produce the name `portal`.

### 3.6 Azure Virtual Desktop and workstations

Users connect from their laptops to WORKSTATION1, WORKSTATION2 and WORKSTATION3. In Azure Virtual Desktop terms these are session host VMs inside the virtual network. The control plane (gateway, broker, web access) is managed by Microsoft and is not part of the customer VNet.

The diagram labels each workstation with a public IP. See [section 11](#11-security-considerations) for why this deserves a review.

### 3.7 Workstations subnet

Holds the three workstation VMs. They act as DNS clients when a user browses to `portal.veltrixrocket.internal`.

### 3.8 Benefits Portal subnet

A separate subnet holds one VM at `10.124.0.4` running nginx as the Benefits Portal web server. Its own subnet lets you manage NSG rules for the portal separately from the workstations.

### 3.9 Resolution chain components

| Component | Role |
|---|---|
| Recursive DNS server | Resolves names for clients. Answers from cache, or queries other servers until it has an answer |
| Root DNS server | Returns a referral to the servers for the requested TLD. Never returns the final address |
| TLD DNS server | Returns a referral to the authoritative name servers for the domain |
| Authoritative DNS server | Holds the zone data and returns the final answer. The diagram's recursive example shows Cloudflare in this role |

## 4. Public DNS Resolution

How an internet user reaches `dashboard.veltrixrockets.space`:

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Client as Browser and OS resolver
    participant Rec as Recursive DNS server
    participant Root as Root server
    participant TLD as space TLD server
    participant Auth as Authoritative server
    participant VM as Dashboard VM 54.56.21.45

    User->>Client: Open dashboard.veltrixrockets.space
    Client->>Client: Check hosts file and local cache
    Client->>Rec: A record for dashboard.veltrixrockets.space?
    Rec->>Rec: Check cache, miss
    Rec->>Root: Where is the space TLD?
    Root-->>Rec: Referral to space TLD servers
    Rec->>TLD: Where is veltrixrockets.space?
    TLD-->>Rec: Referral to the domain's name servers
    Rec->>Auth: A record for dashboard.veltrixrockets.space?
    Auth-->>Rec: 54.56.21.45 and TTL
    Rec-->>Client: 54.56.21.45
    Client->>VM: Connect to 54.56.21.45
```

The server-side steps are described in [section 6](#6-recursive-dns-resolution). What is specific to this design:

- The name servers that give the final answer are the ones listed in the registrar's delegation. If the delegation points to servers that do not host the zone, the lookup fails even though the zone data is correct.
- The answer is a public IP, so the connection to the dashboard VM goes over the internet. Port and protocol are not shown in the diagram.

## 5. Private DNS Resolution

How a workstation resolves `portal.veltrixrocket.internal`.

**Assumption:** the diagram shows no custom DNS servers. The VNet is assumed to use the default Azure-provided DNS at the virtual IP `168.63.129.16`, and the Private DNS zone is assumed to be connected through a virtual network link.

```mermaid
sequenceDiagram
    autonumber
    actor User as Internal user
    participant WS as Workstation
    participant AzDNS as Azure-provided DNS 168.63.129.16
    participant PDZ as Private DNS zone
    participant Web as nginx Benefits Portal 10.124.0.4

    User->>WS: Browse to portal.veltrixrocket.internal
    WS->>WS: Check hosts file and local cache
    WS->>AzDNS: Query portal.veltrixrocket.internal
    AzDNS->>PDZ: Look up record in linked zone
    PDZ-->>AzDNS: 10.124.0.4
    AzDNS-->>WS: 10.124.0.4
    WS->>Web: Connect over the VNet
```

What decides whether this works:

- The workstation's VNet must have a virtual network link to the zone. Without it, Azure DNS returns NXDOMAIN.
- `168.63.129.16` is reachable only from inside Azure. Queries from a laptop or an on-premises network fail.
- If the VNet moves to custom DNS servers, they must forward `veltrixrocket.internal` queries to `168.63.129.16`, or the name stops resolving.
- NSG outbound rules must not block traffic to `168.63.129.16`.

## 6. Recursive DNS Resolution

The lower right of the diagram shows a lookup for `dhondhu.com`. It is a generic example and not a lookup in the public zone (see [section 15](#15-assumptions-and-limitations)).

### 6.1 Client-side order

| Component | Role |
|---|---|
| Hosts file | Static name-to-IP entries. If the name is present, it is used and DNS is not queried |
| Local DNS server | A local caching or stub resolver on the machine or network |
| DNS client | The OS resolver component. Keeps a cache and sends queries to the configured DNS server |

The exact order depends on the OS. On Linux it is set by the `hosts:` line in `/etc/nsswitch.conf`, normally `files` before `dns`. On Windows the DNS Client service checks the hosts file and its own cache before it queries a DNS server. The diagram's label "local DNS server" is ambiguous. In most setups it is a resolver cache on the machine, such as `systemd-resolved`, or a caching forwarder on the network.

### 6.2 Server-side resolution

```mermaid
flowchart TD
    A["Application requests a name"] --> B{"Entry in hosts file?"}
    B -- Yes --> Z["Use hosts file address"]
    B -- No --> C{"Answer in OS DNS client cache?"}
    C -- Yes --> Y["Return cached answer"]
    C -- No --> D["Send query to the configured recursive DNS server"]
    D --> E{"Answer in resolver cache?"}
    E -- Yes --> X["Return cached answer"]
    E -- No --> F["Ask a root server"]
    F --> G["Referral to TLD servers"]
    G --> H["Ask a TLD server"]
    H --> I["Referral to authoritative name servers"]
    I --> J["Ask an authoritative server"]
    J --> K["Answer with TTL"]
    K --> L["Resolver caches the answer and returns it to the client"]
```

| Step | Action |
|---|---|
| 1 | The recursive server checks its cache. A hit is answered immediately |
| 2 | On a miss, it asks a root server about the TLD |
| 3 | The root server returns a referral to the TLD servers |
| 4 | The recursive server asks a TLD server about the domain |
| 5 | The TLD server returns a referral to the domain's authoritative name servers |
| 6 | The recursive server asks an authoritative server for the record |
| 7 | The authoritative server returns the address and a TTL |
| 8 | The recursive server caches the answer for the TTL and returns it to the client |

### 6.3 Recursive and iterative queries

The client sends a **recursive** query: it asks the resolver for the final answer and expects either an answer or an error. The recursive server then sends **iterative** queries to the root, TLD and authoritative servers. Each of them either answers or returns a referral that points closer to the answer, and the recursive server follows the referrals itself.

## 7. Public DNS vs Private DNS

| Aspect | Public DNS zone | Azure Private DNS zone |
|---|---|---|
| Example in this design | `dashboard.veltrixrockets.space` | `portal.veltrixrocket.internal` |
| Answers queries from | Any resolver on the internet | Only linked virtual networks |
| Typically resolves to | Public IPs, such as `54.56.21.45` | Private IPs, such as `10.124.0.4` |
| Delegation | Required: NS records at the registrar | None. No registrar involved |
| Namespace | A registered domain | Any private suffix, here `.internal` |
| Resolution path | Recursive resolver, root, TLD, authoritative | Azure-provided DNS answers from the linked zone |
| Visibility | Records can be queried by anyone | Not visible outside linked VNets |
| Typical use | Internet-facing services | Internal services and VNet-to-VNet traffic |

## 8. Azure Network Architecture

| Property | Value | Basis |
|---|---|---|
| VNet address space | `10.124.0.0/16` | Diagram |
| Subnets | Workstations, Benefits Portal | Diagram |
| Subnet prefixes | Not shown | Assumption, see below |
| Benefits Portal VM | `10.124.0.4`, nginx | Diagram |
| Private DNS record | `portal.veltrixrocket.internal` → `10.124.0.4` | Diagram |
| Workstations | WORKSTATION1, WORKSTATION2, WORKSTATION3, each labelled with a public IP | Diagram |

### 8.1 Why the address is `10.124.0.4`

Azure reserves five addresses in every subnet: the network address (`.0`), the default gateway (`.1`), two addresses for Azure DNS (`.2` and `.3`) and the broadcast address (the last address). The first address a VM can get is therefore `.4`.

`10.124.0.4` fits a subnet whose prefix starts at `10.124.0.0`. It suggests that the Benefits Portal subnet begins at the start of the VNet range and the portal VM holds the first assignable address. This is an inference, since the diagram does not show the subnet prefix.

### 8.2 Example subnet layout

**Recommended:** an example, not taken from the diagram.

| Subnet | Example prefix |
|---|---|
| Benefits Portal | `10.124.0.0/24` |
| Workstations | `10.124.1.0/24` |

### 8.3 Recommended practices

- Give the nginx VM a static private IP on its network interface, because the DNS record depends on `10.124.0.4`.
- Apply NSGs at subnet level. Allow only the Workstations subnet to reach the Benefits Portal subnet on the portal's port.
- Link the Private DNS zone to the VNet. Enable auto-registration only if you want VM name records in this zone.

## 9. Benefits Portal Access Flow

```mermaid
flowchart LR
    U["User laptop"] -->|"AVD connection"| W["Workstation<br/>Workstations subnet"]
    W -->|"1. DNS query<br/>portal.veltrixrocket.internal"| D["Private DNS zone<br/>through VNet link"]
    D -->|"2. Answer 10.124.0.4"| W
    W -->|"3. Request to 10.124.0.4"| N["nginx Benefits Portal<br/>Benefits Portal subnet"]
    N -->|"4. Response"| W
```

1. The user connects from a laptop to one of the three workstations through Azure Virtual Desktop.
2. In the workstation's browser, the user opens `portal.veltrixrocket.internal`.
3. The workstation resolves the name through the VNet's DNS, which answers from the Private DNS zone with `10.124.0.4`.
4. The workstation sends the request to `10.124.0.4` inside the VNet.
5. nginx returns the response over the same private path.

The traffic between the laptop and the workstation is the AVD session. The request to the portal is made from the workstation, not from the laptop, so the laptop never needs to resolve the private name.

**Assumption:** the portal's protocol and port are not shown. **Recommended:** serve the portal over HTTPS, with a certificate from an internal CA or a public CA that can issue for the chosen name.

## 10. DNS Caching and Troubleshooting

### 10.1 Where answers are cached

| Layer | Examples | Notes |
|---|---|---|
| Browser | Chromium and Firefox internal DNS cache | Short-lived, separate from the OS cache |
| OS resolver | `systemd-resolved`, Windows DNS Client | Follows the record TTL |
| Local DNS or stub | dnsmasq, a router | Depends on configuration |
| Recursive resolver | ISP or public resolver | Keeps answers, and negative answers, until the TTL or negative TTL expires |

Each answer carries a TTL, and a resolver keeps the record until it expires, so a changed record may not reach every client straight away. Negative answers such as NXDOMAIN can be cached too, for a time derived from the zone's SOA record. Lowering the TTL before a planned change shortens the period in which clients see the old value.

### 10.2 Hosts file

The hosts file is checked before DNS. An entry like this makes the name resolve locally, whatever DNS returns:

```text
10.124.0.4   portal.veltrixrocket.internal
```

This is handy for testing and a common cause of confusing results. `dig` and `nslookup` query DNS directly and ignore the hosts file, while `getent hosts` follows the system's name-service lookup order, so the tools can return different answers for the same name.

| OS | Hosts file |
|---|---|
| Linux, macOS | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

### 10.3 Linux commands

Useful on the nginx VM, on an admin VM in the VNet, or on any machine for the public names.

| Command | Use it to |
|---|---|
| `nslookup`, `dig`, `host` | Query DNS directly and compare resolvers |
| `dig +trace` | Follow the delegation from the root and find where a public lookup breaks |
| `getent hosts` | See the answer an application gets, including the hosts file and `nsswitch.conf` |
| `resolvectl` | Inspect and flush `systemd-resolved`: DNS servers per interface, queries, caches |

```bash
# Public name, using the system resolver
nslookup dashboard.veltrixrockets.space
dig dashboard.veltrixrockets.space
dig +short dashboard.veltrixrockets.space
host dashboard.veltrixrockets.space

# Compare with a specific public resolver
dig @1.1.1.1 dashboard.veltrixrockets.space A

# Check delegation and the zone's own NS and SOA records
dig veltrixrockets.space NS
dig veltrixrockets.space SOA

# Walk the chain from the root, through the .space TLD, to the authoritative servers
dig +trace dashboard.veltrixrockets.space

# Private name. Run this from a VM inside the linked VNet
dig portal.veltrixrocket.internal
dig portal.veltrixrocket.internal @168.63.129.16

# What applications see: hosts file, nsswitch.conf, then DNS
getent hosts portal.veltrixrocket.internal

# systemd-resolved: configured DNS servers, a query, and the cache
resolvectl status
resolvectl query portal.veltrixrocket.internal
resolvectl flush-caches

# Resolution order and configuration
grep '^hosts:' /etc/nsswitch.conf
cat /etc/hosts
cat /etc/resolv.conf
```

`dig +trace` follows referrals from the root itself, so it shows the delegation as the registry publishes it and does not use the local resolver's cache. It is not useful for `.internal` names because that suffix is not delegated.

### 10.4 Windows commands

**Assumption:** the workstations run Windows, which is typical for AVD session hosts. The diagram does not state the OS.

| Command | Use it to |
|---|---|
| `ipconfig /all` | Show the DNS servers assigned to the adapter |
| `ipconfig /displaydns`, `ipconfig /flushdns` | Inspect and clear the DNS client cache |
| `Resolve-DnsName` | Query DNS from PowerShell, optionally against a chosen server |
| `Test-NetConnection` | Test the TCP connection once the name resolves |

```powershell
# DNS servers used by the workstation
ipconfig /all
Get-DnsClientServerAddress

# Client cache
ipconfig /displaydns
ipconfig /flushdns
Clear-DnsClientCache

# Private name through the configured DNS server, and directly through Azure DNS
nslookup portal.veltrixrocket.internal
Resolve-DnsName portal.veltrixrocket.internal
Resolve-DnsName portal.veltrixrocket.internal -Server 168.63.129.16 -DnsOnly

# Public name
Resolve-DnsName dashboard.veltrixrockets.space -Type A

# Connectivity after resolution. The port is an assumption: use 80 if the portal serves HTTP
Test-NetConnection portal.veltrixrocket.internal -Port 443
```

`nslookup` queries a DNS server directly, so like `dig` it ignores the hosts file.

### 10.5 Troubleshooting checklist

| Symptom | What to check |
|---|---|
| Name does not resolve | The record exists in the correct zone, and the name is spelled correctly |
| Wrong IP returned | A hosts file entry, a stale cache (flush it), or a TTL that has not yet expired |
| Private name fails from a workstation | The virtual network link, the VNet's DNS server setting, and the spelling: `veltrixrocket` (private) vs `veltrixrockets` (public) |
| Private name resolves from outside the VNet | It should not. Check that the record exists only in the Private DNS zone |
| Public name fails from the internet | NS delegation at the registrar, the record in the zone, and `dig +trace` to find the break in the chain |
| Name resolves but the connection fails | NSG rules, nginx listening on the expected port, and the host firewall |
| `dig` and the browser disagree | The hosts file or the browser cache. Compare with `getent hosts` |

## 11. Security Considerations

These are recommendations for this kind of design. They are not statements about what is currently configured.

### 11.1 Public DNS and domain

| Area | Recommendation |
|---|---|
| Registrar account | Enable MFA, the registrar lock and auto-renew |
| Delegation | Watch for unexpected changes to NS records |
| Unused records | Remove records that point to IPs or hosts you no longer control, to avoid subdomain takeover. Review `dhondhu`, whose purpose the diagram does not show |
| DNSSEC | Enable it if the DNS provider supports it |
| CAA records | Limit which certificate authorities may issue certificates for the domain |
| Public visibility | Treat the public zone as readable by anyone. Never put internal names in it |
| Web tier | Serve the dashboard over TLS and consider a WAF or reverse proxy in front of it |

### 11.2 Azure network and Private DNS

| Area | Recommendation |
|---|---|
| Public IPs on workstations | The diagram labels each workstation with a public IP. AVD session hosts do not need an inbound public IP because they connect outbound to the AVD service. Remove the public IPs where possible |
| NSGs | Deny inbound traffic by default. Allow the Workstations subnet to reach the Benefits Portal subnet on the portal port only |
| Administrative access | Avoid RDP or SSH from the internet. Use Azure Bastion or just-in-time VM access |
| Private DNS | Link the zone only to VNets that need it. Limit who can edit it with Azure RBAC |
| Resource locks | Apply locks to the DNS zones and the VNet |
| Sign-in | Use Microsoft Entra ID with MFA and Conditional Access for AVD |
| Encryption | Use HTTPS for the portal even on a private network |
| Logging | Enable NSG flow logs and diagnostic settings, and review Defender for Cloud recommendations |
| Governance | Use Azure Policy to require diagnostics and to deny public IPs on selected resources |

### 11.3 Resolver and client side

- A modified hosts file can redirect traffic without any DNS change. Monitor endpoints for changes to it.
- Recursive resolvers can be targets for cache poisoning. Prefer resolvers that validate DNSSEC, and use encrypted DNS (DoT or DoH) where policy allows.
- Use a dedicated private suffix such as `.internal` for internal zones. A real public domain used internally can collide with, or leak into, the public namespace.

## 12. Technologies Used

| Technology | Role in this design | Basis |
|---|---|---|
| Domain registrar | Domain purchase and NS delegation | Diagram |
| Public DNS zone | Authoritative public records | Diagram |
| Azure Virtual Network | `10.124.0.0/16` private network | Diagram |
| Azure Private DNS | Internal name resolution | Diagram |
| Azure Virtual Desktop | User access to workstations | Diagram |
| Azure virtual machines | Workstations, dashboard VM, portal VM | Diagram |
| nginx | Benefits Portal web server | Diagram |
| Recursive, root, TLD and authoritative DNS servers | Resolution chain | Diagram |
| Cloudflare | Authoritative DNS in the recursive example | Diagram |
| NSGs, Azure Bastion, Azure Policy, Defender for Cloud | Security and governance | Recommended |
| `dig`, `nslookup`, `host`, `getent`, `resolvectl`, `Resolve-DnsName` | Troubleshooting | Documentation |

## 13. Technical Learnings

- Buying a domain and hosting its records are separate steps. NS records at the registrar connect them.
- Public DNS publishes services to the internet. Private DNS keeps internal names and addresses invisible outside the linked VNets.
- Resolution happens in layers. The hosts file, the OS cache, the recursive resolver and the authoritative servers can each hold a different answer at the same time.
- Root and TLD servers return referrals, not answers. The recursive resolver follows them.
- The TTL controls how fast a change reaches clients.
- DNS tools and applications can disagree. `dig` and `nslookup` ignore the hosts file, while `getent hosts` follows the system's lookup order.
- Azure reserves five addresses per subnet, so the first assignable host address is `.4` in a subnet that starts at `.0`. That is why `10.124.0.4` looks the way it does.
- A DNS record that points at an IP makes that IP a dependency. The portal VM should keep a static address.

## 14. Future Improvements

These are proposals, not completed work.

| Improvement | Reason |
|---|---|
| Define the VNet, DNS zones and VMs as Infrastructure as Code (Terraform or Bicep) | Repeatable, reviewable, version-controlled deployments |
| Remove public IPs from the workstations | Smaller attack surface |
| Add Azure Bastion for administrative access | No exposed management ports |
| Serve the portal over HTTPS with an internal CA or managed certificates | Encryption in transit |
| Add a second portal VM behind an internal load balancer | Availability for the Benefits Portal |
| Add Azure DNS Private Resolver | Needed if on-premises networks or other VNets must resolve the private zone |
| Enable DNSSEC and CAA records on the public zone | Better integrity and control over certificate issuance |
| Put a WAF, such as Azure Front Door, in front of the public dashboard | Protection against common web attacks |
| Enable DNS query logging and alerts | Visibility into resolution failures |
| Add automated DNS checks to CI, for example `dig` assertions | Catch record mistakes before users see them |
| Add Azure Policy guardrails | Enforce tagging, diagnostics and network rules |

## 15. Assumptions and Limitations

The diagram is the source of truth for this documentation.

### 15.1 Observations about the diagram

| # | Observation |
|---|---|
| 1 | The private zone is `veltrixrocket.internal` (singular) and the public domain is `veltrixrockets.space` (plural). Both are kept as shown |
| 2 | The recursive example looks up `dhondhu.com`, the browser box shows `google.com`, and the resolved address is `45.45.56.54`. None of these match a record in the public zone, whose addresses start with `45.56.54` and `54.56.21`. The recursive example is treated as a generic illustration |
| 3 | The TLD server in the example is labelled `a.gtld.com`. The real `.com` TLD servers are named `a.gtld-servers.net`, and this project's TLD is `.space`. The label is treated as a simplified example |
| 4 | No DNS provider is shown for the public zone, and no hosting platform is shown for the public VM |
| 5 | Subnet prefixes, NSGs, route tables, DNS server settings and the virtual network link are not shown |
| 6 | The workstations are labelled with public IPs, which AVD session hosts do not require |

### 15.2 Assumptions

- The public records are IPv4 A records.
- The VNet uses the default Azure-provided DNS, and the Private DNS zone is connected through a virtual network link.
- `portal.veltrixrocket.internal` is a manually created A record.
- The portal is reachable from the workstations over HTTP or HTTPS. The port is not specified.
- The workstations are Windows-based AVD session hosts. This is why Windows commands are included.

### 15.3 Not claimed

- This documentation does not claim that the environment is deployed, in production, monitored or load tested.
- No screenshots, metrics or deployment output are included.
- The public IP addresses are used as written in the diagram. They are in publicly routable ranges. Confirm that you control them, or replace them with documentation ranges (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`) if they are placeholders.

## 16. Repository Structure

```text
enterprise-dns-architecture
├── README.md
└── docs
    └── images
        └── architecture-diagram.png
```

---

## 17. Author and Acknowledgements

👤 **Author Aniket Kumar** · DevSecOps Engineer

[![GitHub](https://img.shields.io/badge/GitHub-aniket--devop-181717?style=flat&logo=github&logoColor=white)](https://github.com/aniket-devop)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-aniket484-0A66C2?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/aniket484/)
[![Email](https://img.shields.io/badge/Email-aniketkmr484@gmail.com-D14836?style=flat&logo=gmail&logoColor=white)](mailto:aniketkmr484@gmail.com)

🙏 Thanks to the mentors, learning resources, and open-source tools that supported this project.

📄 Licensed under the [MIT License](LICENSE).

---
