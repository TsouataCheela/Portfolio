# [Tsouata Tchoupou Leslie Cheela] — Cybersecurity & Network Engineering Portfolio

Network and Security Engineer | BTech in Computer Network and System Maintenance, IUGET Bonamoussadi | Douala, Cameroon

📧 [lesliecheela@gmail.com] · 🔗 [https://www.linkedin.com/in/tsouata-cheela-ba982a288] · 💻 [https://github.com/TsouataCheela]

A collection of hands-on infrastructure and security projects I've designed, deployed, and maintained. Each entry breaks down the architecture, the problem, my approach, and what changed after implementation.

---

## 1. Enterprise Privileged Access Management & Zero-Trust Session Auditing
**October 2025 – Present**

**Stack:** WALLIX Bastion, WALLIX Access Manager, Active Directory (LDAP), CIFS storage, OpenZiti overlay network, PNETLab (lab simulation)

### The Architecture

On-prem environment. Around 50 production servers, a mix of Windows and non-Windows. All remote users accessed these targets through the WALLIX Access Manager as the front door. Windows servers are accessed through RDP; anything else through SSH. Connection methods included account method, account mapping, and interactive login.

Session recording went to remote storage over CIFS. Authentication tied back to Active Directory via LDAP — a clean integration, no sync headaches.

Zero-trust networking was handled by OpenZiti, an overlay operating at Layer 3 of the OSI model, working alongside WALLIX Bastion: OpenZiti enforces the zero-trust network layer, while WALLIX Bastion serves as the proxy for session management and password/vault management. The Ziti controller and edge router sat in front of the infrastructure. This wasn't a perimeter-based approach — every connection had to authenticate and be authorised before any packet moved.

### The Problem

A customer (GIE-GCB, part of the GCB banking group) needed this kind of control for security and audit reasons. Shared admin passwords, no session visibility, no granular access control — those were the baseline pain points. I started the whole thing as a proof of concept in PNETLab to validate the design before touching production.

### My Solution

I built out the WALLIX environment step by step:

- Stood up the POC in PNETLab first. Tested LDAP integration, RDP/SSH proxying, session recording, and approval workflows in isolation.
- Deployed the production Bastion version 12.0.9 and Access Manager version 5.1.2. Connected them to the existing AD domain — LDAP queries worked as expected from day one.
- Configured session recording to a dedicated CIFS share. All RDP and SSH sessions were captured and stored remotely, away from the Bastion host itself.
- Implemented password vaulting for privileged accounts. No more static, shared credentials — every access request went through an approval workflow before credentials were released.
- Built the zero-trust overlay using OpenZiti. Installed the controller and edge router, enrolled endpoints, and defined identity-based policies, so that even if someone reached the network, they couldn't talk to a server unless they had an explicit identity and policy allowing that specific connection.

The two things that took real effort: password vaulting and the overlay network setup. Vaulting required careful mapping of which accounts could access which targets, without blocking legitimate admin work. The Ziti controller and edge router configuration had a learning curve — getting identity enrolment and policy syntax right took a few iterations.

### The Outcome

Audits on critical targets increased significantly. Before WALLIX, there was no reliable way to prove who did what. Afterward, every session was recorded, tied to an identity, and reviewable.

Security improved because of the overlay network. Even if an attacker compromised a jump host, they couldn't pivot laterally without passing Ziti's identity checks — the zero-trust layer removed the implicit trust a flat network had before.

---

## 2. Centralised Authentication & Infrastructure Protection via FortiAuthenticator MFA
**October 2025 – Present (proof of concept validated; production rollout at customer site upcoming)**

**Stack:** FortiAuthenticator, FortiGate firewalls, Active Directory (LDAP), WALLIX Bastion, Windows Server (FortiAuthenticator agents), Cisco switches, Huawei switches, Mikrotik routers, FortiToken Mobile, Email OTP

### The Architecture

The goal was simple: stop relying on local usernames and passwords for infrastructure devices. Everything had to authenticate against a central source, and that source had to enforce multi-factor authentication.

FortiAuthenticator sat in the middle. It integrated with:

- **FortiGate** firewalls (RADIUS for admin access)
- **Active Directory** via LDAP (user database)
- **WALLIX Bastion** (RADIUS authentication for PAM sessions)
- **Windows servers** through FortiAuthenticator agents
- **Cisco and Huawei switches** (RADIUS through AAA for CLI access)
- **Mikrotik routers** (RADIUS for management access)

MFA methods used: FortiToken Mobile (TOTP) and Email OTP.

### The Problem

Before this, network devices had a mix of local accounts and AD accounts. Some switches had a shared enable password. Some servers allowed password-only logon. There was no unified authentication policy, and no second factor anywhere. A single compromised password on a switch could be game over.

I needed a way to centralise authentication and add a hard second factor to every critical login, without ripping out the existing directory.

### My Solution

I deployed FortiAuthenticator as the authentication hub as a proof of concept, ahead of a live customer rollout. The design was straightforward: one place to manage users, tokens, and policies.

- Integrated with Active Directory over LDAP. No schema changes, just a service account with read access, so user management stayed in AD while FortiAuthenticator handled MFA and RADIUS/TACACS+ responses.
- Connected FortiGate firewalls to FortiAuthenticator via RADIUS, so admin logins to the firewalls required FortiToken Mobile or Email OTP.
- Configured WALLIX Bastion to use FortiAuthenticator as its RADIUS server, so privileged sessions through the Bastion also needed a second factor.
- Installed FortiAuthenticator agents on Windows servers, forcing MFA at the Windows logon screen, not just for remote access.
- Pointed Cisco and Huawei switches to FortiAuthenticator using RADIUS - AAA, so every CLI login required MFA. Mikrotik routers used the same protocol for the same purpose.


### The Outcome

The proof of concept validated the entire authentication chain end to end: firewalls, switches, routers, Windows servers, and WALLIX PAM sessions all successfully required a second factor through FortiAuthenticator, with no device type left on username-and-password-only access. Local accounts remain configured only for emergency break-glass scenarios, locked down and rarely used.

With the design validated in the lab, production rollout at the customer site is the next phase — moving this same centralised, auditable, second-factor authentication path from proof of concept into live enterprise use.

---

## 3. Three-Tier Hierarchical Campus LAN & Remote Branch Design
**January 2026**

**Stack:** GNS3, Cisco IOS, Three-Tier Hierarchical Model (Core, Distribution, Access layers), VLANs, Inter-VLAN routing, OSPF, EtherChannel, SNMP-based monitoring

### The Architecture

A simulated enterprise network spanning a main campus and multiple remote branches, built around the classic three-tier hierarchical model. The Core layer handled high-speed backbone routing between the campus and branch sites; the Distribution layer enforced policy, aggregation, and inter-VLAN routing; the Access layer connected end devices, segmented by department and function.

### The Problem

An enterprise network carrying high data volumes needs more than raw bandwidth — it needs traffic isolation, redundancy, and centralised visibility across every site. Without deliberate segmentation and dynamic routing, failures and broadcast traffic spread further than they should, and keeping a multi-site network available becomes guesswork.

### My Solution

- Designed the full three-tier topology in GNS3 with Cisco IOS devices, separating Core, Distribution, and Access layers by function.
- Segmented the network into VLANs, with inter-VLAN routing handled at the Distribution layer to enforce strict broadcast domain containment.
- Configured OSPF as the dynamic routing protocol across the campus and remote branch links, so the network could recalculate paths automatically on a link failure.
- Implemented EtherChannel to bundle links between key switches, adding redundancy and extra throughput on the busiest paths.
- Set up SNMP-based monitoring for continuous visibility into interface states and traffic patterns across the topology.

### The Outcome

A production-ready simulation: a highly available topology with deterministic failover paths, strict broadcast domain containment through VLAN design, and optimised inter-site traffic flows through OSPF — a foundational blueprint for secure enterprise scaling, and the same segmentation-and-redundancy thinking that underpins the WALLIX and FortiAuthenticator work above, applied one layer down at the network itself.
