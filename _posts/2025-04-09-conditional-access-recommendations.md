---
title: "The Essential Guide to Microsoft's Conditional Access Recommendations"
description: A consolidated, actionable guide to Microsoft Entra Conditional Access recommendations, with direct links to official documentation and policy guidance.
date: 2025-09-04T19:42:30+02:00
last_modified_at:
categories: [Entra ID, Conditional Access]
tags:
  - Entra ID
  - Conditional Access
---

Microsoft’s Conditional Access documentation is extensive, but the specific recommendations are scattered across multiple articles and sections, making it difficult to get a consolidated view. This article compiles all of Microsoft’s explicit Conditional Access recommendations from official documentation into one structured, actionable reference. Each recommendation includes a direct link to the source material for further context and detail.

## Design Guidelines
This section covers on design principles, operational resilience, policy rollout, and session management — as distinct from specific policy templates, which are covered in the next section.

### Design principles 
**Source:** [Plan a Conditional Access deployment](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access#recommendations)

- Use a structured naming standard that includes sequence numbers and has descriptive policy names.
- Ensure that every app has at least one policy applied.
- Minimize the number of policies by grouping users and apps covered by repetitive patterns.
- Use protected action as another layer of security on attempts to create, modify or delete policies.
- Implement policy assignments through groups, not individuals.<sup>[1](#ref1)</sup> 
- Ensure a consistent experience across Microsoft 365 client applications by implementing the same set of controls for services such as Exchange Online and SharePoint.<sup>[1](#ref1)</sup> <sup>[2](#ref2)</sup>

> To protect Conditional Access assignment groups, consider using [role-assignable groups](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/groups-concept#how-are-role-assignable-groups-protected) and/or placing them in [restricted-management administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/admin-units-restricted-management) to prevent unauthorized modification.
{: .prompt-info }

### Operational Resilience
**Source:**  [Create a resilient access control management strategy](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-resilient-controls#microsoft-recommendations)

- Provide multiple authentication methods for each user, using different communication channels.
- Combine multiple Conditional Access controls to avoid lockouts:
  - Use Windows Hello for Business to satisfy MFA requirements at sign-in.
  - Use trusted devices (hybrid-join or Intune-managed) to satisfy strong authentication requirements without implicit MFA challenge.
  - Leverage risk-based policies that prevent access when the user or sign-in is at risk, in place of fixed MFA policies.
- Do regular reviews of the exception groups.<sup>[1](#ref1)</sup>
- Deploy password hash sync even if federated or using pass-through auth.
- Maintain at least two cloud-only emergency permanent-admin accounts using *.onmicrosoft.com domain, phishing-resistant authentication (different to that of other admin accounts).<sup>[3](#ref3)</sup>
- Implement disabled policies that act as secondary resilient access controls in outage or emergency access scenarios.<sup>[4](#ref4)</sup>
- If using on-prem VPN with NPS extension, federate as a SAML app or plan how to disable or replace in an emergency. 

### Policy Deployment Process
**Source:** [Plan a Conditional Access deployment](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access#deploy-conditional-access-policies)

- Use [Microsoft's policy templates](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policy-common?tabs=secure-foundation#template-categories) to get started with a secure baseline and expand from there.
- Evaluate impact with:
  - [Report-only mode](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-report-only) 
  - [What If Tool](https://learn.microsoft.com/en-us/entra/identity/conditional-access/what-if-tool)
- Test thoroughly, report-only does not always match the actual behavior.

> Consider using a phased rollout approach when deploying Conditional Access in production. There is no official guidance on this, but it can help reduce risk.
{: .prompt-info }

### Session and Re-authentication Settings
**Source:** [Reauthentication prompts and session lifetime](https://learn.microsoft.com/en-us/entra/identity/authentication/concepts-azure-multi-factor-authentication-prompts-session-lifetime#configure-settings-for-microsoft-entra-session-lifetime)

- Enable single sign-on across applications by using either:
  - [Managed devices](https://learn.microsoft.com/en-us/entra/identity/devices/overview) 
  - [Seamless SSO](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-sso)
- If reauthentication is needed, configure a [sign-in frequency policy](https://learn.microsoft.com/en-us/entra/identity/conditional-access/howto-conditional-access-session-lifetime).
- For mobile users, encourage the use of the Microsoft Authenticator app to reduce authentication prompts.
- Avoid KMSI (Keep Me Signed In/Company Branding)- use Conditional Access to enforce persistent browser sessions instead.


### Deprecation and Transition Guidance

- [Migrate legacy risk policies](https://learn.microsoft.com/en-us/entra/id-protection/howto-identity-protection-configure-risk-policies#migrate-risk-policies-to-conditional-access) from Entra ID Protection to Conditional Access.
  - Legacy risk policies retire on October 1, 2026.
- [Migrate to the Authentication Methods policy](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods-manage#migration-between-policies) from Legacy MFA (per-user MFA) and SSPR policies.
  - Legacy MFA/SSPR deprecation begins September 30, 2025.

<hr>

## Recommended Policies

The ['Secure foundation'](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-policy-common?tabs=secure-foundation#tabpanel_1_secure-foundation) templates provide the minimum baseline that all organizations should implement first. These policies drive broad MFA adoption and device-based authentication, establishing essential security controls before advanced protections are implemented.

![recommended policies from microsoft documentation]({{site.baseurl}}/assets/img/2025-04-09-ca-recommendations/ca-doc.png){: .normal }

Once that baseline is in place, you can move toward more advanced protections. Focus on risk-based controls, phishing-resistant authentication for admins, authentication strength, and alignment with the [Zero Trust model](https://learn.microsoft.com/en-us/security/zero-trust/deploy/identity).

The recommended policies below support this progression. They introduce more granular and risk-aware access controls.

### Secure foundation

- [Block legacy authentication for all users](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-legacy-authentication)
- [Securing security info registration](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-security-info-registration)
  - It’s recommended to use Temporary Access Pass (MFA) for registration of internal users.
  - For guest users who need to register MFA in your directory, registration can be blocked outside a trusted network location.
- [Require device compliance](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-device-compliance)

> Device compliance may be challenging to enforce in practice - see [alternatives to device compliance below](#alternative-to-device-compliance).
{: .prompt-info }

### Zero Trust

- [Require MFA authentication strength for all users](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-mfa-strength)
  - Do not include any app exclusions for this policy, see '[Conditional Access behavior when an all resources policy has an app exclusion](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-cloud-apps#conditional-access-behavior-when-an-all-resources-policy-has-an-app-exclusion)'.
- [Require MFA authentication strength for guests](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-guests-mfa-strength)
- [Require MFA for risky sign-in](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-risk-based-sign-in)
- [Require password change for risky users](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-risk-based-user)
  - For passwordless accounts It’s recommended to BLOCK users at high risk.

> Authentication strength is not supported for external users who authenticate with email OTP, SAML/WS-Fed or Google federation. 
{: .prompt-warning }

### Emerging threats

- [Require phishing resistant MFA for administrators](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-admin-phish-resistant-mfa)

### Uncategorized

- [Require authentication strength for device registration](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-device-registration)
  - External authentication methods are currently incompatible with authentication strength (use MFA grant control instead)
  - To enforce CA user action 'Register or join devices', ensure the corresponding MFA requirement in 'Devices > Overview > Device Settings' is disabled.
- [Restrict device code flow and authentication transfer](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-authentication-flows)
- [Block disallowed countries/regions](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-block-by-location) (Recommended in '[Plan your CA deployment](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access#recommendations)') 

### Alternatives to Device Compliance <a id="alternative-to-device-compliance" href="#"></a>

- [Require compliant or hybrid joined device for admins](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-alt-admin-device-compliand-hybrid)
- [Require compliant, hybrid joined device OR MFA for all users](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-alt-admin-device-compliand-hybrid)
- [Block unknown or unsupported device platforms](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-device-unknown-unsupported)
- [Disable browser persistence](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-persistent-browser)
  - Prevents persistent browser sessions from unmanaged devices.

<hr> 

## Other Considerations

While the following topics are not explicitly listed as part of Microsoft's conditional access recommendations, they represent practical and security-relevant scenarios that many organizations should consider:

- [Conditional Access for workload identities](https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity) to protect service principals. 
- [Conditional Access authentication context for PIM Role activation](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/just-in-time-access-to-groups-and-conditional-access-integration-in-privileged-i/2466926) to ensure elevated access is subject to just-in-time controls.
- Reduce attack surface by monitoring and cleaning up [inactive users](https://learn.microsoft.com/en-us/entra/id-governance/create-access-review#create-a-single-stage-access-review), [stale guests accounts](https://learn.microsoft.com/en-us/entra/identity/users/clean-up-stale-guest-accounts) and [inactive workload identities](https://techcommunity.microsoft.com/blog/microsoft-entra-blog/new-app-health-recommendations-in-microsoft-entra-workload-identities/2959984).
- Use [macOS Platform Single Sign-on](https://learn.microsoft.com/en-us/entra/identity/devices/macos-psso) to enable strong, seamless authentication on macOS- similar to how WHfB is used on Windows devices.
- Configure [Single Sign-On for Linux](https://learn.microsoft.com/en-us/entra/identity/devices/sso-linux?tabs=debian-install%2Cdebian-update%2Cdebian-uninstall) to improve authentication on supported distributions.
- [Prevent token replay attacks](https://learn.microsoft.com/en-us/entra/identity/devices/protecting-tokens-microsoft-entra-id#token-theft--protect-against-replay) with [Conditional Access Token Protection](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection#requirements)


### License requirements

The table below summarizes license dependencies for features discussed in this article. Refer to the [Microsoft Entra licensing](https://docs.azure.cn/en-us/entra/fundamentals/licensing) for more details.

| **License Tier**           | **Required For**                                                                                   |
|----------------------------|----------------------------------------------------------------------------------------------------|
| **Microsoft Entra ID P1** | - Conditional Access |
| **Microsoft Entra ID P2** | - Risk-based policies <br> - Access Reviews (group membership) <br> - Privileged Identity Management (PIM) |
| **Entra ID Governance**   | - Access Reviews (Inactive Users) |
| **Workload ID Premium**   | - Conditional Access for Workload IDs (service principals) | 


<hr>

## What's Next?

Conditional Access is evolving rapidly. These recommendations provide a solid foundation, but successful implementation requires structured policy design and consistent review.

For practical guidance, see [Building Scalable Conditional Access – A Policy Framework for Zero Trust](https://alflokken.github.io/posts/conditional-access-framework/). It covers naming, layering, and organizing policies for complex environments.

### Further reading

**Microsoft:**
- [Common security policies for Microsoft 365 organizations](https://learn.microsoft.com/en-us/security/zero-trust/zero-trust-identity-device-access-policies-common) (App protection and device compliance)
- [Conditional Access optimization agent](https://learn.microsoft.com/en-us/entra/identity/conditional-access/agent-optimization) ("copilot" agent for CA)
- [Protecting tokens in Microsoft Entra](https://learn.microsoft.com/en-us/entra/identity/devices/protecting-tokens-microsoft-entra-id) (harden devices and CA token protection)
- [Conditional Access insights and reporting](https://learn.microsoft.com/en-us/entra/identity/conditional-access/howto-conditional-access-insights-reporting)
- [Developer guidance for Microsoft Entra Conditional Access](https://docs.azure.cn/en-us/entra/identity-platform/v2-conditional-access-dev-guide)

**Other resource:**

- [Conditional Access Essentials: RMAUs, Named Locations, Authentication Strengths, Service Principals](https://www.welkasworld.com/post/conditional-access-essentials-rmaus-named-locations-authentication-strengths-service-principals) (excellent article with details on protecting groups)

## References

<a id="ref1" href="#">1</a>: [Microsoft Entra authentication management operations reference guide](https://learn.microsoft.com/en-us/entra/architecture/ops-guide-auth#conditional-access-implementation) <a href="#design-principles">🔗</a> <br>
<a id="ref2" href="#">2</a>: [Service dependencies in Microsoft Entra Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/service-dependencies) <a href="#design-principles">🔗</a> <br>
<a id="ref3" href="#">3</a>: [Manage emergency access accounts](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/security-emergency-access) <a href="#operational-resilience">🔗</a> <br>
<a id="ref4" href="#">4</a>: [Plan a Conditional Access deployment - Naming standards for emergency access controls](https://learn.microsoft.com/en-us/entra/identity/conditional-access/plan-conditional-access#naming-standards-for-emergency-access-controls) <a href="#operational-resilience">🔗</a> <br>

## Changelog
- 2025-09-12: Updated the “What’s Next?” section to include a link to the CA Framework guide.
- 2025-09-04: Initial version.