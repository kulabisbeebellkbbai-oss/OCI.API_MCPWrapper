# OCI API MCP Wrapper — Approved Project Summary

## Intended outcome

Build a loopback-only Model Context Protocol (MCP) server that enables other
agentic systems on this machine to use the complete Oracle Cloud Infrastructure
(OCI) API, including read, write, and destructive operations.

## Primary users

The primary users are local agentic systems and the humans supervising their
work.

## Major components

- A Streamable HTTP MCP server bound only to `127.0.0.1`.
- Safe discovery and selection of existing local OCI CLI profiles.
- Python OCI SDK integration for typed service operations.
- A validated generic signed-request capability for OCI operations not covered
  by typed tools, preserving complete API coverage.
- Operation discovery and schema or parameter guidance.
- Pagination, retry handling, OCI work-request tracking, and structured errors.
- Explicit confirmation controls for destructive and high-impact operations.
- Redacted audit logging that never records credential material.
- Health checks, automated tests, installation support, local MCP registration,
  and operator documentation.

## Likely risks

- Destructive, irreversible, or billable cloud operations.
- Exposure of OCI credentials, signing keys, profile contents, or sensitive
  response data.
- The breadth and continuing evolution of the OCI API surface.
- IAM policy and permission differences between OCI profiles.
- Free Tier service limits, quotas, regional availability, and accidental use
  of billable resources.
- Long-running operations, pagination, throttling, retries, and partial
  failures.
- Malicious or mistaken endpoint, header, and payload inputs.

## Constraints

- Provide complete OCI API coverage, including destructive operations.
- Authenticate through existing local OCI CLI profiles.
- Restrict MCP access to loopback clients on this machine.
- Never commit, return, or log OCI credentials, private keys, or unredacted
  profile material.
- Keep destructive and high-impact calls available but require explicit
  confirmation before execution.
- Validate profiles, endpoints, headers, and payloads and maintain redacted
  audit records.
- Keep live testing within the designated OCI Free Tier tenancy and avoid
  unintended charges.

## Deliverables

- A functional loopback-only OCI MCP server.
- Complete-coverage generic API access plus safer typed support tools.
- Automated unit, schema, transport, and security tests.
- Live OCI tenancy integration tests wherever Free Tier availability and IAM
  permissions safely permit them.
- Installation, configuration, MCP registration, operation, maintenance, and
  security documentation.
- Any local service and health integration required for reliable operation.

## Immediate next steps

The first milestone will:

- start the loopback MCP server;
- complete MCP initialization, tool discovery, and tool-call flows;
- enumerate OCI profiles without exposing secrets;
- authenticate with a selected profile;
- perform harmless tenancy, region, and compartment reads;
- execute one generic signed read request; and
- prove that an unconfirmed destructive request is blocked and audited.

After this approved summary is saved, invoke Superpowers brainstorming and
complete its design approval gates. Then invoke Superpowers writing-plans and
save the authoritative implementation plan under `docs/superpowers/plans/`.

## Approval

Approved by the user on 2026-08-02.
