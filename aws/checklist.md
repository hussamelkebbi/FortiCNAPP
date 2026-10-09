# FortiCNAPP AWS onboarding: SCP and RCP prerequisites

Use this checklist before deploying **Configuration (CSPM) and organization CloudTrail** integrations. The customer's AWS organization policy owner should review the policies and approve any required exceptions.

**SCPs limit what deployment and ingestion identities can do. RCPs can restrict access to member-account resources, including access by external FortiCNAPP identities.** Successful resource creation alone does not confirm FortiCNAPP can assume the integration role.

## Customer review checklist

| Area | Required configuration | Verification / action |
| --- | --- | --- |
| **Policy scope** | Include every applicable SCP and RCP. | Inspect policies attached to the organization Root, every parent OU, and each target account. Discover policy IDs; do not rely on policy names. |
| **Approved region** | Deployment is permitted in **`<APPROVED_AWS_REGION>`**. | Review region-based denies, including `aws:RequestedRegion`. Use an approved region or a scoped exception. |
| **Deployment actions** | The onboarding identity can perform the actions required by the selected FortiCNAPP deployment policies. | Check CloudFormation, IAM, and supporting services used by the selected modules, such as Lambda, SNS, Secrets Manager, S3, SQS and KMS. Review explicit denies and any SCP allow-list restrictions. Do not grant unrestricted service access merely because a service appears here. |
| **IAM role creation** | Integration roles and their policies can be created and maintained. | Check restrictions on `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PutRolePolicy`, and related update/deletion actions required by the deployment. |
| **Mandatory IAM controls** | The deployment satisfies required role paths, tags and permissions boundaries. | Provide these requirements before deployment. Configure supported module inputs or adapt the templates to comply. |
| **External role assumption** | RCPs permit the verified FortiCNAPP principal to call `sts:AssumeRole` on the intended integration roles. | Review denies on `sts:*` or `sts:AssumeRole`, especially conditions restricting callers to the customer's organization. Add a scoped exception where required. |
| **CloudTrail ingestion** | Applicable policies permit the configured ingestion role to read logs, consume notifications and decrypt data as required. | Review S3, SQS and KMS restrictions together with bucket, queue and key policies. Separately verify AWS CloudTrail can deliver logs and notifications. |
| **Change ownership** | Policy changes are made through the customer's policy-management process. | Identify the policy owner and source of truth, including Control Tower or other automation. Preserve unrelated controls. |

> **An explicit Deny overrides an Allow, including AdministratorAccess.** Adding another Allow does not resolve an explicit deny. The request must satisfy the existing controls, or the blocking deny must be adjusted.

## Which role needs the RCP exception?

| Identity | Location | Purpose |
| --- | --- | --- |
| **Deployment identity** | Customer management account or approved delegated administrator | Runs the onboarding deployment. It is separate from the external FortiCNAPP identity. |
| **FortiCNAPP principal** | Vendor AWS account | Assumes the customer's integration role. The template inspected for this deployment specifies **`arn:aws:iam::434813966438:role/lacework-platform`**. Verify the principal in the customer's chosen integration template before applying an exception. |
| **Configuration integration role** | Customer target account | Grants FortiCNAPP access for configuration assessment. In the inspected deployment, the name was **`cnapp-laceworkcwsrole-sa`**; this name depends on the configured prefix. |

The name `lacework-platform` comes from the vendor principal in the integration template. It is **not a role the customer must create**.

## Example: exception to an organization-only RCP deny

In the diagnosed deployment, an RCP denied `sts:*` to principals outside the customer's organization. The role and external ID were correct, but the RCP prevented FortiCNAPP from assuming the member-account role.

For a matching deny, the following JSON fragment can be added **alongside the existing conditions in that Deny statement** to exempt the verified vendor principal:

```json
"ArnNotEquals": {
  "aws:PrincipalArn": "arn:aws:iam::434813966438:role/lacework-platform"
}
```

**This is a policy fragment, not a complete policy or a shell command.** Preserve the other conditions and statements. If the condition already exists, merge carefully rather than replacing its existing exceptions.

The exception applies to **all actions and resources covered by that statement**, across its attachment scope. If approval covers only STS access to selected roles, split or narrow the statement so the exception has that scope. Do not remove the entire RCP or exempt all external principals.

The exception does not grant access by itself. The integration role must still trust the correct FortiCNAPP principal, require the correct external ID, and have the required permissions. Other applicable policies continue to apply.

## Completion criteria

| Check | Required evidence |
| --- | --- |
| **Pilot deployment** | One representative member account for each distinct policy configuration deploys and registers successfully. |
| **Organization rollout** | Every expected member StackSet instance reports **`CURRENT` / `SUCCEEDED`**. Check the management-account deployment separately. |
| **Configuration integration** | Every expected account is registered in FortiCNAPP and assessment begins successfully. |
| **CloudTrail integration** | The organization trail is logging; member-account logs arrive in the central bucket; member-account events are searchable in FortiCNAPP. |

**A successful Terraform apply alone is insufficient:** StackSet failure-tolerance settings can allow an operation to finish while individual member stacks fail.

## References

- [AWS service control policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)
- [AWS resource control policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html)
- [AWS guidance on restricting external access with RCPs](https://aws.amazon.com/blogs/security/effectively-implementing-resource-controls-policies-in-a-multi-account-environment/)
- [CloudFormation StackSets concepts and operation status](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html)
- [Lacework organization configuration Terraform module](https://registry.terraform.io/modules/lacework/org-configuration/aws/latest)
