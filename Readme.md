# AWS SSO setup and application integration

> AWS renamed **AWS Single Sign-On (AWS SSO)** to **AWS IAM Identity Center**. This guide uses the current service name. It covers (1) workforce access to AWS accounts and (2) connecting a separate application to AWS for single sign-on. These are related but distinct configurations.

## 1. Understand the sign-in flow

Typical setup with a corporate identity provider (IdP), such as Microsoft Entra ID or Okta:

1. The user opens the AWS access portal or the application.
2. AWS IAM Identity Center redirects the browser to the configured IdP.
3. The IdP authenticates the user (password, MFA, conditional access, etc.) and returns a signed SAML response to AWS or to the application.
4. IAM Identity Center matches the user to a provisioned identity, checks assignments, and grants the configured AWS account role or application access.
5. For an application, the application creates its own session after validating the SAML response. AWS does not automatically create application accounts unless the application supports provisioning (usually SCIM) or the app creates users on first login.

**SAML** carries the authentication result and selected attributes. **SCIM** provisions and updates users/groups; it does not authenticate users. In an external-IdP setup, the IdP is normally the source of truth for identities.

## 2. Choose the setup pattern

- **AWS account access:** IAM Identity Center assigns users/groups to AWS accounts with permission sets. Users then obtain short-lived AWS access through the access portal or AWS CLI.
- **Third-party application access:** Configure the application as a SAML service provider (SP) in IAM Identity Center, then assign users/groups to that application.
- **Both:** Use one organization instance and configure both account access and application access as needed.
- **No corporate IdP / lab:** Use the built-in Identity Center directory to create test users and groups. For a production workforce, connect the existing IdP where possible.

The organization instance is the recommended option for centralized multi-account access and customer-managed applications. An account instance has a narrower feature set; check AWS's [instance comparison](https://docs.aws.amazon.com/singlesignon/latest/userguide/organization-instances-identity-center.html) before choosing.

## 3. Prerequisites and planning

Before changing AWS settings, collect:

- An AWS Organizations management account (recommended for an organization instance) and an administrator permitted to configure IAM Identity Center.
- The AWS Region where IAM Identity Center will run. An organization can have its instance in only one primary Region; changing Regions later requires deleting/recreating the instance, so decide deliberately.
- Your IdP administrator and application administrator, if different people.
- The IdP's supported federation method (SAML 2.0; SCIM 2.0 for automated user/group provisioning where supported).
- For each application: SP metadata XML or exact **ACS/Reply URL**, **Entity ID/Audience URI**, required SAML claim names, and whether it supports IdP-initiated and/or SP-initiated login.
- A pilot group, at least one test user, an application test account, and a break-glass AWS administrator path that does not depend on the new SSO configuration.

Agree on a stable user key before configuring anything. The IdP SAML `NameID` used to sign in must match the username identity provisioned to IAM Identity Center (commonly email/UPN, depending on your directory and app). Confirm exact casing, format, and uniqueness with the app owner.

## 4. Enable IAM Identity Center

1. Sign in to the AWS Organizations **management account** with an administrator identity. Avoid using root for routine work.
2. Open the [IAM Identity Center console](https://console.aws.amazon.com/singlesignon/).
3. Choose the intended Region and enable IAM Identity Center as an **organization instance**. Review instance and encryption options before confirming. Multi-Region/customer-managed KMS options have additional prerequisites and cost implications.
4. In **Settings → Identity source**, decide which source owns identities:
   - Keep **Identity Center directory** for a small lab or manually managed users.
   - Connect the existing external IdP (for example Entra ID or Okta) if that is the organization's source of truth. Follow the vendor-specific AWS tutorial for its current SAML settings.
   - Use Active Directory only if that is the chosen directory design; configure synchronization scope and directory connectivity.
5. Do not switch identity sources casually. AWS warns that a source change can remove users, groups, assignments, and permission set roles. Plan and test migrations before changing it.
6. Record the access portal URL and confirm an administrator can reach it.

AWS setup reference: [Enable IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-identity-center.html).

## 5. Add identities and groups

### Built-in directory

1. In IAM Identity Center, open **Users** and create a pilot user with username, given name, family name, display name, and unique email.
2. Create groups that reflect access boundaries (for example `App-Name-Users`, `AWS-Dev-ReadOnly`).
3. Add the pilot user to the appropriate group(s). Store and distribute initial sign-in instructions securely.

### External IdP

1. Configure the IdP connection using the matching AWS guide (for example [Microsoft Entra ID](https://docs.aws.amazon.com/singlesignon/latest/userguide/idp-microsoft-entra.html) or [Okta](https://docs.aws.amazon.com/singlesignon/latest/userguide/gs-okta.html)). SAML enables sign-in; it does not by itself create the users/groups in IAM Identity Center.
2. Either provision users/groups manually in IAM Identity Center or enable SCIM if both sides support it.
3. For SCIM, in IAM Identity Center **Settings**, enable automatic provisioning, then copy the SCIM endpoint and access token into the IdP provisioning configuration. Treat the token as a secret; AWS displays it at creation, so store it securely. Rotate it if exposed or nearing expiry.
4. Map required attributes and assign the correct users/groups to the IdP's IAM Identity Center enterprise app. Start with the pilot group and verify create, update, group membership, and deactivation behavior.
5. Ensure the SAML `NameID` and SCIM username refer to the same user identity. AWS documents this as a key requirement for successful sign-in.

See AWS [SCIM provisioning considerations](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html). Provisioning frequency and deactivation behavior depend on the IdP.

## 6. Give users AWS account access (if needed)

This section is for AWS console/CLI access, not access to an unrelated application.

1. In IAM Identity Center, create a **permission set** for each job function. Grant only the permissions needed; prefer AWS managed or carefully reviewed customer-managed policies, and use permission boundaries/session duration as appropriate.
2. Open **AWS accounts**, choose the target account(s), and assign the pilot group/user plus the permission set.
3. Wait for assignments to provision, then use the access portal to verify the user sees only the expected account and role.
4. Test AWS CLI access using the supported IAM Identity Center login flow (`aws configure sso`, then `aws sso login --profile <profile>`). Use short-lived credentials; do not distribute long-term access keys for human users.
5. Add production groups/accounts only after the pilot succeeds. Review access periodically and remove assignments when no longer required.

## 7. Connect a SAML application to IAM Identity Center

First obtain the SP values from the application's SSO/admin page. Do not guess the ACS URL or audience; exact values are application-specific.

1. In the AWS console, open **IAM Identity Center → Applications → Customer managed → Add application**.
2. Choose **I have an application I want to set up**, then **SAML 2.0**.
3. Enter a clear display name and description.
4. Download the IAM Identity Center **SAML metadata XML** and signing certificate. Provide these to the application administrator. These identify AWS as the IdP to the application; protect them and track certificate renewal.
5. In **Application metadata**, enter the application's exact **Application ACS URL** and **Application SAML audience** (SP Entity ID). If the app supplied SP metadata, use it to populate/confirm these values.
6. Set optional start URL, relay state, and session duration only if the application requires them and your policy allows the requested session length.
7. Save the application.
8. Open its details and configure **Attribute mappings** to match the app's documentation. Common mappings include the app's username/email claim to the corresponding Identity Center email or username attribute, plus first name, last name, and groups/roles only where explicitly supported. Use the exact attribute URI/name and format the SP expects.
9. Assign a pilot user/group to the application. An unassigned user may authenticate successfully at the IdP but still be denied by IAM Identity Center.
10. In the application, configure AWS as the trusted IdP by uploading/importing the AWS metadata XML or entering its issuer, SSO URL, and certificate as required. Enable SAML login and map received claims to the application's user identity/roles.
11. Test both SP-initiated login (start at the app) and IdP-initiated launch (start at the access portal) if the application supports them. Some apps support only one path.

AWS's [custom SAML application procedure](https://docs.aws.amazon.com/singlesignon/latest/userguide/customermanagedapps-set-up-your-own-app-saml2.html) and [attribute mapping instructions](https://docs.aws.amazon.com/singlesignon/latest/userguide/mapawsssoattributestoapp.html) are the source of truth for console labels and mappings.

### What the application administrator must configure

Send the application team the AWS IdP metadata XML (preferred) or the issuer/entity ID, SSO URL, and signing certificate from IAM Identity Center. Ask them to return/confirm their ACS URL, SP Entity ID/audience, required claims, supported login-initiated flows, and whether they support SCIM. They must configure AWS as a trusted IdP, validate the SAML signature, map a stable NameID to an app account, and define how groups/roles authorize users. SAML login proves identity; the application still decides what the user may do.

### Optional: provision application users with SCIM

SAML app integration does not automatically provision accounts inside the application. If the application supports SCIM, configure its provisioning connector separately using the app's SCIM endpoint and credential/token, map attributes and groups, and test create/update/deactivate in a pilot. Confirm which system owns group membership and how deprovisioning behaves. Do not reuse the IAM Identity Center inbound SCIM token for an unrelated application's SCIM endpoint.

## 8. How to troubleshoot the sign-in exchange

1. Identify which hop failed: app → IAM Identity Center → corporate IdP → IAM Identity Center → app ACS. Record timestamp, user, browser, and correlation/request ID; do not send passwords, tokens, or full assertions in tickets.
2. Verify the user exists in IAM Identity Center, is active, has required attributes, and is assigned to the app. For external users, verify SCIM/IdP sync completed.
3. Compare exact ACS URL, audience/Entity ID, issuer, and recipient/destination values on both sides. A URL typo, trailing slash, wrong environment, or stale metadata commonly breaks trust.
4. Check the `NameID` format/value matches the provisioned username and app's expected account key. Check every required claim name, case, value, and format.
5. Confirm the AWS signing certificate/metadata is current in the app, and the app signing certificate/metadata is current in AWS if configured. Renew and coordinate certificate changes before expiry.
6. Confirm browser cookies/pop-up/redirect behavior and whether the failed test used SP-initiated or IdP-initiated flow.
7. Review IAM Identity Center and IdP audit/sign-in logs, plus app authentication logs. An authentication success followed by an app 403 usually points to missing app assignment, account mapping, or authorization/group mapping.

## 9. Production readiness checklist

- [ ] IAM Identity Center Region, instance type, and identity source are documented.
- [ ] Pilot SSO succeeds for assigned users; unassigned users are denied.
- [ ] MFA and conditional-access rules are enforced by the corporate IdP where appropriate.
- [ ] Least-privilege permission sets and app roles are reviewed by their owners.
- [ ] Joiner/mover/leaver process is tested, including app and AWS account deprovisioning.
- [ ] SCIM tokens and signing certificates have named owners, secure storage, expiry monitoring, and rotation steps.
- [ ] Break-glass administrator access and recovery procedure are tested and monitored.
- [ ] Audit logs and support ownership are known; users know where to report sign-in failures.
- [ ] Configuration values are stored in an approved secrets/configuration location, not committed to source control.

## Official AWS references

- [Enable IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-identity-center.html)
- [Organization instances](https://docs.aws.amazon.com/singlesignon/latest/userguide/organization-instances-identity-center.html)
- [Connect an external identity provider](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
- [Set up a custom SAML 2.0 application](https://docs.aws.amazon.com/singlesignon/latest/userguide/customermanagedapps-set-up-your-own-app-saml2.html)
- [Map application attributes](https://docs.aws.amazon.com/singlesignon/latest/userguide/mapawsssoattributestoapp.html)
- [Provision users/groups with SCIM](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)
