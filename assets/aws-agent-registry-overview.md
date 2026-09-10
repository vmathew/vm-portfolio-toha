# AWS Agent Registry — Overview

*Compiled September 10, 2026*

**AWS Agent Registry** is a private, governed catalog for your organization's AI agents, MCP servers, tools, and skills. It gives teams one place to publish, approve, find, and reuse them.

- **Preview:** April 2026, inside Amazon Bedrock AgentCore
- **General availability:** August 31, 2026, with its own `agent-registry` namespace

---

## The problem it solves

The problem is **agent sprawl**. Once dozens of teams are building agents and MCP servers, nobody knows what already exists, who owns it, whether it's approved, or whether the team down the hall built the same thing. That creates compliance gaps and wasted effort. The registry fixes the discovery and governance side of that.

---

## Core concepts

### Registry

A registry is a catalog you create in your AWS account. Each one has:

- A name and a description
- An **authorization configuration**, which controls how consumers reach the discovery APIs and the MCP endpoint
- An **approval configuration**, which sets whether records need manual review or are approved automatically

Common ways to organize registries: by resource type, by environment (prod, QA, dev), by team or business unit, or by organizational scope.

### Registry record

A record describes one resource. Each record has:

- `name`: required. The name plus `recordVersion` must be unique within the registry.
- `displayName` and `description`: optional
- `recordVersion`
- `recordType`: `AGENT`, `MCP`, `SKILL`, or `CUSTOM`
- A descriptor specific to the type

### Record types

| Type | What it describes | Format / validation |
|---|---|---|
| **AGENT** | An autonomous agent's capabilities and skills | A2A (Agent-to-Agent protocol) agent card, validated against the A2A schema |
| **MCP** | An MCP server and the tools it exposes (inputs and outputs) | MCP server definition, validated against the MCP schema |
| **SKILL** | A reusable capability shared across agents | Name and description, optional package or repo link, optional markdown docs |
| **CUSTOM** | Anything else: APIs, Lambda functions, knowledge bases, databases | Any valid JSON |

The agents and servers you register don't have to run on AWS. They can run on another cloud or on-premises.

---

## Record lifecycle and approval

```
Draft → Submitted → Approved ─→ Deprecated
                 └→ Rejected → (revise and resubmit)
```

- **Draft:** the publisher is still working on it
- **Submitted:** waiting for a curator
- **Approved:** visible to consumers through search, browse, and the MCP endpoint
- **Rejected:** goes back to the publisher to fix
- **Deprecated:** removed from discovery because it was retired or replaced

**Approval workflow**

- Approval can be manual, or automatic (for example, in dev registries).
- Submitting a record sends an Amazon EventBridge event, which you can route to ticketing systems or existing review pipelines.
- Curators record their decision with the `UpdateRegistryRecordStatus` API.

---

## Discovery (consumer side)

Consumers only see **approved** records.

| Method | APIs | Notes |
|---|---|---|
| **Search** | `SearchDiscoverableRegistryRecords` | Hybrid search: natural language ("find a tool that can book flights") plus exact keywords ("weather-api-v2"). Filter by name, type, or version. |
| **Browse** | `ListDiscoverableRegistryRecords`, `GetDiscoverableRegistryRecord`, `BatchGetDiscoverableRegistryRecord` | Paginated list with filters |
| **MCP endpoint** | `InvokeRegistryMcp` | The registry acts as an MCP server that IDEs and agents can query directly, without the AWS SDK. Integrates with Kiro and Amazon Quick. |

---

## Security and access

**Inbound (how consumers get in)**

- **IAM:** the simplest option for teams already on AWS
- **JWT:** tokens from your corporate identity provider (Amazon Cognito, Okta, Azure AD, or other OAuth 2.0 providers)
- Admin (control-plane) operations **always** use IAM.

**Outbound (record sync)**

- A record can point to a live MCP or A2A URL and pull in updated metadata automatically: server name, description, and tools. Each sync creates a new revision of the record.
- The registry uses a **credential provider** (OAuth credentials or an IAM role ARN) to call those URLs.

---

## Roles

| Role | What they do |
|---|---|
| **Administrator** | Creates and configures registries, sets authorization and approval rules, manages IAM, and has full CRUD access to records |
| **Publisher** | Creates records, refines metadata, submits for approval, and sets up URL sync |
| **Curator / Approver** | Reviews submissions against security, compliance, and metadata standards, then approves, rejects, or deprecates |
| **Consumer** | Finds and uses approved records through search, browse, or the MCP endpoint |

---

## Enterprise features

- **Org-wide auto-detection:** works with AWS Organizations to find AgentCore Runtimes and Gateways across member accounts automatically, with no per-account setup. Auto-detected records are labeled as such and linked back to their source.
- **Cross-account sharing:** share registries across accounts with AWS RAM (Resource Access Manager).
- **Audit:** CloudTrail logs every control-plane API call as a management event.
- **Infrastructure as code:** CloudFormation, Terraform, and AWS CDK.
- **Tags:** on both registries and records, for cost allocation, access control, and tracking.

---

## Preview → GA migration (time-sensitive)

At GA, the registry moved out of `bedrock-agentcore` into its own namespace.

| Surface | Preview (old) | GA (new) |
|---|---|---|
| CLI | `aws bedrock-agentcore` | `aws agent-registry` |
| Data plane endpoint | `bedrock-agentcore.{region}.amazonaws.com` | `agent-registry.{region}.api.aws` |
| Control plane endpoint | `bedrock-agentcore-control.{region}.amazonaws.com` | `agent-registry-control.{region}.api.aws` |
| IAM action prefix | `bedrock-agentcore:*` | `agent-registry:*` |
| Service principal | `bedrock-agentcore.amazonaws.com` | `agent-registry.amazonaws.com` |
| ARN prefix | `arn:aws:bedrock-agentcore:...:registry/...` | `arn:aws:agent-registry:...:registry/...` |
| Managed policy | `BedrockAgentCoreFullAccess` | `AgentRegistryFullAccess` |
| SDK clients | `BedrockAgentCoreClient` / `...ControlClient` | `AgentRegistryClient` / `AgentRegistryControlClient` |
| EventBridge source | `aws.bedrock-agentcore` | `aws.agent-registry` |
| CloudWatch namespace | `AWS/BedrockAgentCore` | `AWS/AgentRegistry` |

**API schema changes**

- Records now require `name` and `recordType`. The old `name` field became `displayName`.
- `descriptorType` → `recordType`, `inlineContent` → `data`, and `schemaVersion`/`protocolVersion` → `dataSchemaVersion`.
- `synchronizationConfiguration` → `source`, which now sits inside each descriptor.
- `SearchRegistryRecords` → `SearchDiscoverableRegistryRecords`. List operations changed from GET with query parameters to POST with structured `filters`.
- `autoApproval` (boolean) → `autoApprovalRules` (enum array). Authorization settings moved under `discoveryConfiguration`.

**Key dates**

- **Aug 6, 2026:** the new namespace launched with migration tooling. Both namespaces work in parallel.
- **Sep 17, 2026:** the old namespace shuts down, and **data left in it is lost**.

**How to migrate**

- Migration is manual. Use the migration tool in the [agentcore-samples](https://github.com/awslabs/agentcore-samples) repo. You can run it locally or in CloudShell, run it on AWS Glue through a CDK stack, or write to both namespaces in parallel while you validate.
- Update IAM trust policies for synced records to the new service principal.
- **Don't** migrate permissions for workload identity or OAuth credential providers. Those stay in `bedrock-agentcore`.

---

## Availability and pricing

- **Regions:** US East (N. Virginia), US West (Oregon), Europe (Ireland), Asia Pacific (Tokyo), Asia Pacific (Sydney)
- **Pricing:** not listed in the GA announcement or the docs. Check the AgentCore pricing page.

## Similar products

- Microsoft: Entra Agent Registry and Azure Agent Registry
- Google Cloud: Agent Registry
- ACP (Agent Client Protocol) Registry, a registry defined at the protocol level

Early coverage says AWS's version stands out for indexing agents that run anywhere, using MCP and A2A. Early customers named in coverage include Southwest Airlines and Zuora.

---

## Sources

- [AWS Agent Registry docs (overview)](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry.html)
- [Concepts and terminology](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-concepts.html)
- [Key capabilities](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-key-capabilities.html)
- [Get started](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-get-started.html)
- [Migration guide / FAQ](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/registry-faq.html)
- [GA announcement (Aug 2026)](https://aws.amazon.com/about-aws/whats-new/2026/08/aws-agent-registry-generally-available/)
- [Preview announcement (Apr 2026)](https://aws.amazon.com/about-aws/whats-new/2026/04/aws-agent-registry-in-agentcore-preview)
- [InfoQ: AWS Launches Agent Registry in Preview](https://www.infoq.com/news/2026/04/aws-agent-registry-preview/)
- [The Register: AWS built a registry](https://www.theregister.com/2026/04/09/aws_ai_agent_registry/)
