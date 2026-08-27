# Engineering Portfolio

A collection of hands-on infrastructure and security projects I’ve designed, deployed, and maintained. Each entry breaks down the architecture, the problem, my approach, and what changed after implementation.

---

## 1. Enterprise Privileged Access Management & Zero-Trust Session Auditing

**Stack:** WALLIX Bastion, WALLIX Access Manager, Active Directory (LDAP), CIFS storage, OpenZiti overlay network, PNETLab (lab simulation)

### The Architecture

On-prem environment. Around 50 production servers, mix of Windows and non-Windows. All remote users accessed these targets through the WALLIX Access Manager as the front door. Windows servers are accessed through RDP; anything else is through SSH. Connection methods included account method, account mapping, and interactive login.

Session recording went to remote storage over CIFS. Authentication tied back to Active Directory via LDAP—clean integration, no sync headaches.

Zero-trust networking was handled with OpenZiti, an overlay operating at Layer 3. The Ziti controller and edge router sat in front of the infrastructure. This wasn’t a perimeter-based approach. Every connection had to authenticate and be authorized before any packet moved.

### The Problem

A customer (GIE-GCB) needed this kind of control for security and audit reasons. Shared admin passwords, no session visibility, no granular access control—those were the baseline pain points. I started the whole thing as a proof of concept in PNETLab to validate the design before touching production.

### My Solution

I built out the WALLIX environment step by step:

- Stood up the POC in PNETLab first. Tested LDAP integration, RDP/SSH proxying, session recording, and approval workflows in isolation.
- Deployed the production Bastion and Access Manager. Connected them to the existing AD domain. No issues there—LDAP queries worked as expected from day one.
- Configured session recording to a dedicated CIFS share. All RDP and SSH sessions were captured and stored remotely, away from the Bastion host itself.
- Implemented password vaulting for privileged accounts. No more static, shared credentials. Every access request went through an approval workflow before credentials were released.
- Built the zero-trust overlay using OpenZiti. Installed the controller and edge router, enrolled endpoints, and defined identity-based policies. This meant that even if someone reached the network, they couldn’t talk to a server unless they had an explicit identity and policy allowing that specific connection.

The two things that took real effort: password vaulting and the overlay network setup. Vaulting required careful mapping of which accounts could access which targets, and making sure the workflow didn’t block legitimate admin work. The Ziti controller and edge router configuration had a learning curve—getting the identity enrollment and policy syntax right took a few iterations. Both problems were solved, but not without some late nights.

### The Outcome

Audits on critical targets increased significantly. Before WALLIX, there was no reliable way to prove who did what. Afterward, every session was recorded, tied to an identity, and reviewable.

Security improved because of the overlay network. Even if an attacker compromised a jump host, they couldn’t pivot laterally without passing Ziti’s identity checks. The zero-trust layer removed the implicit trust that a flat network had before.

---

## 2. Centralized Authentication & Infrastructure Protection via FortiAuthenticator MFA

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

Before this, network devices had a mix of local accounts and AD accounts. Some switches had a shared enable password. Some servers allowed password-only logon. There was no unified authentication policy, and definitely no second factor. A single compromised password on a switch could be game over.

I needed a way to centralize authentication and add a hard second factor to every critical login, without ripping out the existing directory.

### My Solution

I deployed FortiAuthenticator as the authentication hub. The design was straightforward: one place to manage users, tokens, and policies.

- Integrated with Active Directory over LDAP. No schema changes, just a service account with read access. That kept user management in AD while FortiAuthenticator handled MFA and RADIUS/TACACS+ responses.
- Connected FortiGate firewalls to FortiAuthenticator via RADIUS. All admin logins to the firewalls now required FortiToken Mobile or Email OTP.
- Configured WALLIX Bastion to use FortiAuthenticator as its RADIUS server. So any privileged session through the Bastion also needed a second factor.
- Installed FortiAuthenticator agents on Windows servers. This forced MFA at the Windows logon screen, not just for remote access.
- Pointed Cisco and Huawei switches to FortiAuthenticator using TACACS+. Every CLI login—enable mode included—required MFA. Mikrotik routers used RADIUS for the same purpose.

No major issues during this deployment. The trickiest part was making sure the various device types all sent the correct RADIUS or TACACS+ attributes, but once the first switch was working, the rest were repetitive.

### The Outcome

Infrastructure devices no longer accept a simple username and password. Every admin login—firewall, switch, router, server, PAM—now goes through the same central MFA policy. That alone eliminates the risk of a single stolen password granting direct network access.

Local accounts still exist on some devices for emergency break-glass, but they’re locked down and rarely used. The day-to-day authentication path is centralized, auditable, and requires a second factor.
