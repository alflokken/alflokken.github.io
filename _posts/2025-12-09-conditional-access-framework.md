---
title: Building Scalable Conditional Access - A Policy Framework for Zero Trust
description: A practical framework for building scalable Conditional Access policies in Microsoft Entra ID, with structured naming, layered design, and Zero Trust alignment.
date: 2025-09-12T07:32:30+02:00
categories: [Entra ID]
tags:
  - Entra ID
  - Conditional Access
  - Zero Trust
  
---

Conditional Access policies are a core enforcement mechanism in Microsoft Entra ID's [Zero Trust](https://learn.microsoft.com/en-us/azure/security/fundamentals/zero-trust#zero-trust-architecture) model, where every access request must be explicitly verified. However, without a structured approach, Conditional Access policies often evolve reactively, becoming difficult to manage, inconsistent, and prone to gaps in coverage.

Most organizations start without a structured plan, leading to overlapping policies, unclear naming, and uncertainty about what applies in a given scenario.

This article introduces a Conditional Access framework designed to address these challenges. It builds on Claus Jespersen's [Conditional Access Principles and guidance](https://github.com/microsoft/ConditionalAccessforZeroTrustResources/blob/main/ConditionalAccessGovernanceAndPrinciplesforZeroTrust%20October%202023.pdf), refines the naming model and aligns it with current Entra ID capabilities.

> <sub>Microsoft recommends <a href="https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access#set-naming-standards-for-your-policies" style="color: inherit !important;">establishing a meaningful naming standard</a>, but their examples don’t scale well in complex environments:</sub>
> ![microsoft ca naming example]({{site.baseurl}}/assets/img/2025-12-09-caframework/ms-ca-example.png){: .normal }

This framework replaces vague policy names like 'guest access' with structured documentation. For example, 'CA300-Guests-Base-AllApps-AnyContext-MFA' clearly indicates that the policy (CA300) applies to guest users and enforces their baseline (default) level of protection, requiring multi-factor authentication for all applications in any context.

## Key Concepts

This framework uses two core elements:

* Structured naming — policies are organized by:
  * Persona: who the policy applies to
  * Purpose: what the policy enforces

* Layered targeting — policies apply at multiple levels.
  * Baseline policies define minimum controls for each persona.
  * Additional policies extend the baseline to cover specific risks or scenarios.

This structure improves manageability and scalability through predictable policy categories. It also supports Zero Trust by ensuring each access request is explicitly verified.

<hr>

## Design Principles

### 1. Layered Defense

Policies apply at multiple levels: global policies for universal requirements like MFA, persona-based baselines that add role-appropriate controls such as phishing-resistant authentication for admins, and purpose-driven policies that address specific risks, such as enforcing additional controls for sensitive applications.

![layered defence illustration]({{site.baseurl}}/assets/img/2025-12-09-caframework/layered_defence.png){: .normal }

<br>

Example: A developer accessing Azure DevOps inherits:
* the global MFA baseline
* internal user controls 
* application-specific controls

**Benefit:** Layered policies allow new applications and users to automatically inherit protections, while every access request is explicitly verified with requirements that scale based on risk.

### 2. Predictable Naming

Policy names should act as compressed documentation. A structured format makes intent clear at a glance.

![CA Components]({{site.baseurl}}/assets/img/2025-12-09-caframework/ca-components.png){: .normal }

**Example:** "CA200-Internals-Base-O365-Any-MFA+Compliant" tells you:

* Who: Internal users
* What: Baseline protection
* Where: Office 365
* When: Any access context
* How: Requires MFA and device compliance

**Benefit:** Predictable naming improves scalability through structured, consistent policies, while also making management and troubleshooting more efficient.

<hr>

## Components

The framework organizes policies using six components that define who they target, what they enforce, and how they are scoped.

They are meant as a starting point, adapt them to fit your environment by adding personas, simplifying categories, or adjusting application scopes. The key is consistent application across your policy set.

| Component | Description | Examples |
|-----------|-------------|---------|
| **Identifier** | Unique policy ID, grouped by persona | CA001, CA100, CA200 |
| **Persona** | Identity category which the policy applies to | Global, Admins, Internals, Guests |
| **Purpose** | Security objective, what the policy protects | Base, Identity, AttackSurface, Data |
| **Scope** | Resource that the policy applies to | AllApps, O365, PrivilegedAccess |
| **Context** | Access method (how the access is being made) | AnyContext, Browser, Mobile, LegacyAuth |
| **Control** | Access Requirement or Enforcement action | MFA, Compliant, Block |
| **Notes** | Description of the policy intent (optional) | "Require passkey for internal users" |

**Example Policies:**

* `CA001-Global-Base-AllApps-AnyContext-MFA`
  * Require MFA for all users across all applications  
* `CA201-Internals-App-O365-AnyContext-Compliant`
  * Require compliant device for internal users accessing Office 365  
* `CA404-Workloads-Identity-Terraform-OutsideTrustedZone-Block - only allow connections from hcp`
  * Block terraform workloads outside allowed named location (with a comment in the notes section)

<hr>

### 1. Identifier

A structured numbering scheme enables logical grouping and easy reference. Instead of saying "that guest MFA policy for SharePoint," an admin can simply refer to "CA305."

* CA001–099: Global policies (apply to all users)  
* CA100–199: Admin-specific policies  
* CA200–299: Internal users  
* CA300–399: Guests or external users
* CA400–499: Workload identities

### 2. Persona

Define personas by targeting well-defined user groups, not individuals, in line with best practices. You can define additional personas to reflect specific roles or risk profiles in your environment, such as Developers, ExternalAdmins, or Vendors.

> To protect these groups, consider using role-assignable groups and/or placing them in restricted-management administrative units to prevent unauthorized modification.
{: .prompt-info }

| Persona | Description | Use Case |
|---------|-------------|----------|
| **Global** | All users in the organization | Baseline security requirements that apply universally |
| **Admins** | Privileged users with elevated access | Stricter controls for high-risk accounts |
| **Internals** | Employees and trusted contractors | Standard enterprise protections |
| **Guests** | External users and partners | Limited access with additional scrutiny |
| **Workloads** | Workloads and service accounts | App-specific authentication requirements |

### 3. Purpose

Specifies the intended security outcome of the policy. This categorization helps organize policies by distinct security objectives.

| Purpose | Description | Examples |
|---------|-------------|----------|
| **Base** | Core protection for each persona group | MFA requirements, device compliance |
| **Identity** | Advanced identity protection | Risk-based access, privileged account protection |
| **AttackSurface** | Reduce potential attack vectors | Block legacy protocols, geographic restrictions |
| **App** | Application-specific controls | Sensitive app protections, API access rules |
| **Data** | Information protection controls | DLP integration, classification-based access |
| **Compliance** | Regulatory requirements | Terms of use, audit logging |

### 4. Scope

Specifies which resources the policy applies to. Grouping related resources helps reduce the number of policies while maintaining comprehensive coverage.

* `AllApps`: Universal application coverage  
* `O365`: Microsoft 365 suite  
* `PrivilegedAccess`: Administrative portals and tools  
* `SensitiveApps`: High-value business applications  
* `SSPR`: Self-service password reset  
* `SecurityInfo`: Security information registration  

### 5. Context

Defines the conditions under which the policy is triggered, based on how and from where the access attempt is made. This could include the client app type, device state, network location, or other contextual factors.

Examples:

* `AnyContext`: All access methods  
* `UnmanagedDevice`: Non-corporate devices
* `Browser`: Web-based access  
* `Mobile`: Mobile device access  
* `LegacyAuth`: Legacy authentication protocols
* `OutsidePerimeter`: Outside 'perimeter' named location
* `Mobile+UnmanagedDevice`: Mobile, not managed by intune.
* `Desktop+Browser`: Web-browser on desktop devices.


### 6. Control

Defines the requirements that must be met before access is granted, such as a combination of grant or session controls.

Common controls:

* `MFA`: Multi-factor authentication required  
* `Compliant`: Device must be compliant with policy  
* `Block`: Access is denied  
* `AppProtection`: Intune app protection policy
* `MFA+Compliant`: Both MFA and device compliance required
* `MFAorCompliant`: MFA or compliant device required
* `Hybrid`: Require hybrid Azure AD joined device  
* `AuthStrength`: Require a specific authentication strength (e.g., phishing-resistant MFA)  

<hr>

## Implementation Strategy

If you're starting fresh, review the conditional access best practices and [policy recommendations](https://alflokken.github.io/posts/conditional-access-recommendations/#recommended-policies) in particular. For organizations with existing deployments, this framework can help rationalize and modernize your current policy set.

### Existing Tenants

1. Audit existing policies by mapping who they target and what controls they apply.
2. Map policies to the framework through categorization, revealing duplicates and inconsistencies.
3. Refactor by applying the naming conventions and removing overlaps.

<hr>

## What's Next?

This framework provides structure and consistency, but effective Conditional Access implementation requires ongoing refinement and disciplined adoption.

To maximize value:

* Adopt deliberately. The framework only delivers results when consistently applied across your policy set.
* Continuously evolve. Threats, users, and business applications change. So should your policies.
* Move beyond the baseline. Once core protections are in place, strengthen your posture with:
  * Phishing-resistant MFA
  * Risk-based access controls
  * Device authentication with Intune or hybrid joined devices

> Identity, and by extension, Conditional Access, is just one pillar of the [broader Zero Trust architecture](https://learn.microsoft.com/en-us/security/zero-trust/deploy/overview). Achieving end-to-end security requires balanced attention to all Zero Trust pillars, as each reinforces the others to build a resilient security posture.

**Further reading:**

* [The Essential Guide to Microsoft's Conditional Access Recommendations](https://alflokken.github.io/posts/conditional-access-recommendations/). 
* [Microsoft's Zero Trust Workshop](https://microsoft.github.io/zerotrustassessment/docs/intro).
* [Get started with phishing-resistant passwordless authentication deployment in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/authentication/how-to-plan-prerequisites-phishing-resistant-passwordless-authentication)