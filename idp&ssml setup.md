# Identity Provider (IdP) and SAML SSO Setup with AWS IAM Identity Center

This guide covers connecting a workforce identity provider (IdP) to AWS IAM Identity Center, synchronizing users, and configuring a SAML application. The examples use Microsoft Entra ID; Okta and other SAML 2.0/SCIM providers follow the same exchange of metadata, but their console labels and steps differ.

## 1. Understand the protocols and sign-in flow

- **SAML 2.0** handles browser authentication. The IdP authenticates the person and returns a signed assertion to the service provider (SP), which in this guide is IAM Identity Center or a SAML-enabled application.
- **SCIM 2.0** provisions, updates, and deactivates users and groups. It does not authenticate users.
- An SAML assertion does not automatically create an IAM Identity Center user. Users/groups must be provisioned manually or through SCIM before they can be assigned.

For AWS access, the flow is: user opens AWS access portal -> IAM Identity Center redirects to the corporate IdP -> the IdP authenticates and sends a SAML response back to IAM Identity Center -> IAM Identity Center matches the user, checks assignments, and grants the configured account/role or application. AWS permissions come from assignments and permission sets, not from successful authentication alone.

## 2. Plan before changing the identity source

Have these items ready:

- AWS administrator access to the IAM Identity Center **organization instance** and access to the IdP admin console.
- A pilot user and group, with a known sign-in username/email.
- A stable user identifier that matches across systems. For IAM Identity Center federation, AWS requires the SAML `NameID` to use email-address format and exactly match the provisioned IAM Identity Center username.
- A break-glass administrative path and a record of existing users, groups, account assignments, and permission sets.
- If integrating an app: its exact ACS/Reply URL, SP Entity ID/Audience URI, required SAML claim names, and supported login-initiated flow(s).

> **Change warning:** Changing an existing IAM Identity Center identity source can remove users, groups, assignments, and permission-set roles. Review AWS's [change identity source considerations](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-change.html), plan recovery, and schedule the change. Do not use this walkthrough as a casual toggle in a production instance.

## 3. Connect an external IdP for AWS workforce access

The SAML trust is configured on both sides: AWS provides service-provider (SP) metadata to the IdP, and AWS imports IdP metadata to trust the IdP.

### Step 3.1: Get AWS service-provider details

1. Sign in to the AWS console with an administrator and open **IAM Identity Center -> Settings -> Identity source**.
2. Choose **Actions -> Change identity source**, select **External identity provider**, and continue to the configuration page. Do not submit the change until both sides are ready.
3. Download the IAM Identity Center **service provider metadata** file, or record the displayed issuer and ACS/sign-in URL values required by your IdP.
4. Keep this page available while configuring the IdP. The metadata may include multiple ACS URLs (for IPv4/dual-stack endpoints and additional Regions). Use the values supported by your IdP and selected AWS Region setup.

### Step 3.2: Configure the AWS enterprise application in the IdP

In Microsoft Entra admin center (menu names can change):

1. Go to **Identity -> Applications -> Enterprise applications** and add/select **AWS IAM Identity Center** from the gallery.
2. Open **Single sign-on -> SAML**.
3. Import/upload the AWS service-provider metadata from Step 3.1 when supported. Otherwise enter the AWS **Identifier (Entity ID/issuer)** and **Reply URL (ACS URL)** exactly as provided by AWS.
4. Set the **Sign-on URL** to the AWS access portal sign-in URL if the IdP setup requests it.
5. Configure the Name ID claim to emit the pilot user's email/username in the required email-address format. Ensure it is the same exact value as the user's IAM Identity Center `Username` (including casing/aliases as applicable).
6. Download the IdP **federation metadata XML** (or obtain issuer, SSO URL, and signing certificate). Keep the signing certificate current and note its expiration.
7. Assign only the pilot user/group to this enterprise application while testing.

For a different IdP, use its AWS IAM Identity Center integration guide. Do not copy ACS URLs or claim names from another tenant/provider without checking the actual metadata and configuration.

### Step 3.3: Import IdP metadata into AWS and complete the change

1. Return to the IAM Identity Center identity-source setup page.
2. Under **Identity provider metadata**, upload the IdP federation metadata XML. AWS uses it to identify the IdP and validate signed SAML responses.
3. Review the values, read the change warning, and confirm only when ready. AWS may ask you to type `ACCEPT` before applying the source change.
4. Confirm the identity source now shows the external IdP and that the IdP configuration shows the AWS application as successfully configured.

AWS's general [external IdP connection procedure](https://docs.aws.amazon.com/singlesignon/latest/userguide/how-to-connect-idp.html) is authoritative for current console labels and metadata requirements.

## 4. Provision users and groups with SCIM (recommended where supported)

SCIM keeps the Identity Center directory aligned with the IdP. If SCIM is not supported, create and maintain users/groups manually in IAM Identity Center.

1. In IAM Identity Center, open **Settings -> Identity source / Automatic provisioning** and enable automatic provisioning. Copy the SCIM endpoint and generate/copy the bearer token as instructed.
2. In the IdP's AWS IAM Identity Center enterprise app, open its provisioning settings, choose **Automatic** provisioning, and enter the SCIM tenant URL/endpoint and bearer token.
3. Test the connection, enable provisioning, and configure attribute mappings for users and groups. Map the IdP attribute used by the SAML Name ID to IAM Identity Center `userName`/Username.
4. Assign the pilot user and group for provisioning, then verify they appear in **IAM Identity Center -> Users/Groups** with the expected username, email, and membership.
5. Test a controlled attribute update, group membership change, and deactivation. Confirm expected downstream access removal before broad rollout.
6. Store the SCIM token as a secret, restrict access, monitor expiry, and rotate it according to AWS/IdP instructions. Never commit it to source control or paste it into a ticket.

If users/groups are provisioned manually, create the matching IAM Identity Center records before making account/application assignments. Avoid making conflicting manual changes to SCIM-managed data because this can cause directory drift.

## 5. Assign AWS access and validate workforce SSO

1. Create least-privilege permission sets for the required job roles.
2. In **IAM Identity Center -> AWS accounts**, assign the pilot group/user to a test account and the appropriate permission set.
3. Open the AWS access portal using the pilot user's IdP account. Confirm the user is redirected to the IdP, signs in, returns to the portal, and sees only the expected account/role.
4. Test AWS CLI access using `aws configure sso` followed by `aws sso login --profile <profile>`.
5. Confirm an unassigned user is denied. Expand assignments only after the pilot passes.

## 6. Configure a separate SAML application (optional)

This is a different trust relationship: here **IAM Identity Center is the IdP** and the third-party application is the SP. SSO into AWS does not automatically configure SSO into another application.

### Collect application SP values

From the application administrator, obtain the exact **ACS/Reply URL**, **SP Entity ID/Audience**, required attribute/claim names and formats, and whether the app supports SP-initiated login, IdP-initiated launch, or both. Also confirm if the app supports SCIM; app provisioning is configured separately.

### Add the application in IAM Identity Center

1. Open **IAM Identity Center -> Applications -> Customer managed -> Add application**.
2. Choose **I have an application I want to set up -> SAML 2.0**.
3. Enter the app's exact ACS URL and audience/Entity ID. Do not guess these values.
4. Download the IAM Identity Center IdP metadata XML (and signing certificate if required) and give it to the app administrator through an approved channel.
5. Configure **Attribute mappings** to the app's documented claim names. Map a stable username/email and add name or group/role claims only where the application requires them.
6. Save the app and assign the pilot user/group.

### Configure the application and test

1. In the application SSO settings, import AWS metadata or configure the AWS issuer, SSO URL, and signing certificate as required by that product.
2. Enable SAML login and map the received NameID/claims to the app's user account and authorization model.
3. Test SP-initiated login from the app and IdP-initiated launch from the AWS access portal if both are supported.
4. Verify that an unassigned Identity Center user cannot launch the app and that app authorization matches the intended role/group.
5. If the app supports SCIM, configure its provisioning connector separately using the app's own endpoint and credential. Do not reuse the IAM Identity Center inbound SCIM token for another service.

See AWS [custom SAML application setup](https://docs.aws.amazon.com/singlesignon/latest/userguide/customermanagedapps-set-up-your-own-app-saml2.html) and [attribute mapping](https://docs.aws.amazon.com/singlesignon/latest/userguide/mapawsssoattributestoapp.html).

## 7. Troubleshooting checklist

| Symptom | Checks |
| --- | --- |
| IdP login succeeds but AWS rejects the user | User exists and is active in IAM Identity Center; SAML NameID exactly equals its Username; email-format NameID; user/group is assigned. |
| SAML response/recipient error | Compare issuer, ACS URL, audience, recipient, and destination values against current metadata. Check region and IPv4/dual-stack URL selection. |
| Signature/certificate error | Refresh IdP metadata/certificate on the AWS side; check signing certificate validity and rollover on both sides. |
| User authenticates but sees no AWS account/app | Verify provisioning completed and the user/group has an account permission-set or application assignment. |
| User is missing or stale | Check SCIM connector status, IdP app assignment, attribute mappings, provisioning logs, and group push/membership rules. |
| App login returns 403 after SAML | Check app account mapping, app assignment, required claims, and the app's own role/authorization rules. |

Use IAM Identity Center, IdP, and application audit logs with timestamps and correlation IDs. Do not share passwords, SCIM bearer tokens, private keys, or full SAML assertions in logs/tickets.

## 8. Production checklist

- [ ] Pilot SAML login succeeds and `NameID` matches the provisioned username exactly.
- [ ] SCIM create/update/deactivate and group membership behavior has been tested, or a manual lifecycle owner is documented.
- [ ] Least-privilege permission sets and application roles are assigned to approved groups.
- [ ] IdP MFA/conditional access, break-glass access, and recovery procedures are documented.
- [ ] SAML signing certificate and SCIM token have owners, secure storage, expiry monitoring, and rotation steps.
- [ ] Unassigned users are denied; joiner/mover/leaver access removal has been verified.
- [ ] Application SAML metadata/claims and AWS account assignments have named owners.

## Official AWS references

- [Connect an external identity provider](https://docs.aws.amazon.com/singlesignon/latest/userguide/how-to-connect-idp.html)
- [External identity provider overview](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
- [AWS and Microsoft Entra ID SAML/SCIM tutorial](https://docs.aws.amazon.com/singlesignon/latest/userguide/idp-microsoft-entra.html)
- [SAML/SCIM federation requirements](https://docs.aws.amazon.com/singlesignon/latest/userguide/other-idps.html)
- [Automatic user/group provisioning with SCIM](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)
- [Set up a custom SAML application](https://docs.aws.amazon.com/singlesignon/latest/userguide/customermanagedapps-set-up-your-own-app-saml2.html)
