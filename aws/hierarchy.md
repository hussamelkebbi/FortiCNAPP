# AWS organization: authentication, deployment, and SCPs

Authentication establishes **who is making the request**. IAM permissions and SCPs determine **whether the requested action is allowed**. Selecting the organization root as a deployment target does not bypass permissions.

```mermaid
flowchart TB
    USER["Operator / Terraform / FortiCNAPP"] --> AUTH{"AWS credentials valid?"}
    AUTH -->|"No — expired or invalid"| AUTHFAIL["Authentication fails<br/>Refresh credentials"]
    AUTH -->|Yes| MGMT["Management account · 788826923680<br/>IAM role permissions authorize deployment"]
    MGMT --> SS["Service-managed CloudFormation StackSet<br/>Target: r-m9s0"]
    SS --> EXEC["StackSets execution roles<br/>in the four member accounts"]
    EXEC --> CHECK{"IAM permits action<br/>and applicable SCPs permit it?"}
    CHECK -->|Yes| SUCCESS["Member stack can deploy"]
    CHECK -->|"No — explicit SCP deny"| DENIED["Member stack fails<br/>AccessDenied: cloudformation:CreateStack"]

    subgraph ORG["Organization · o-qktrht78qg"]
        ROOT["Root · r-m9s0<br/>All five accounts are directly beneath it"]
        ROOT --> MA["Management account<br/>788826923680<br/>SCPs do not restrict this account"]
        ROOT --> MEMBERS["Four member accounts<br/>cnapp-security · 895770859678<br/>cnapp-workload-2 · 510735888512<br/>First Principles · 416086757138<br/>Identity Delegated Admin · 593901683388"]
        ROOT -. "Optional hierarchy — no OUs currently shown" .-> OU["Organizational unit · ou-…"]
        OU -.-> FUTURE["Accounts inside that OU"]
    end

    ROOTSCP["SCP attached to Root"] -. "Inherited by all member accounts" .-> MEMBERS
    ROOTSCP -. "Also inherited through OUs" .-> FUTURE
    OUSCP["SCP attached to an OU"] -. "Applies within that OU" .-> FUTURE
    MEMBERS -. "Applicable SCPs constrain execution roles" .-> CHECK

    classDef blue fill:#dbeafe,stroke:#2563eb,color:#172554;
    classDef green fill:#dcfce7,stroke:#16a34a,color:#14532d;
    classDef red fill:#fee2e2,stroke:#dc2626,color:#7f1d1d;
    classDef amber fill:#fef3c7,stroke:#d97706,color:#78350f;
    classDef gray fill:#f1f5f9,stroke:#64748b,color:#0f172a;
    class USER,MGMT,SS,EXEC,ROOT,MEMBERS blue;
    class SUCCESS green;
    class AUTHFAIL,DENIED red;
    class AUTH,CHECK,ROOTSCP,OUSCP amber;
    class MA,OU,FUTURE gray;
```

## What happened in this deployment

| Check | Observed result |
| --- | --- |
| Authentication | The deployment reached AWS and attempted member-account stack creation. |
| Organization targeting | The StackSet targeted the four member accounts under `r-m9s0`. |
| Member-account authorization | The region restriction SCP `p-eyznnlju` explicitly denied `cloudformation:CreateStack` in `us-east-1`. |
| Management-account deployment | It could succeed because SCPs do not restrict the management account; IAM permissions still apply. |
| Overall verification | Check every expected StackSet instance. A top-level success alone does not establish coverage of every account. |

The SCP failure above describes the earlier deployment. The policy was subsequently reported deleted; a new deployment must be checked for its current result.

**AdministratorAccess cannot override an explicit SCP deny.** SCPs limit permissions; they do not grant credentials or permissions. SCPs can also be attached directly to member accounts. Service-managed StackSets do not deploy stack instances to the management account, so management-account resources require separate deployment.
