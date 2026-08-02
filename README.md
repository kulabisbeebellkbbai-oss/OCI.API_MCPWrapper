# OCI API MCP Wrapper

OCI API MCP Wrapper is a planned loopback-only Model Context Protocol server
for controlled access to the complete Oracle Cloud Infrastructure API through
allowlisted local OCI CLI profiles.

The approved design combines typed discovery and common operations with a
validated generic signed-request fallback. Every state-changing operation uses
a separate approval flow, with human approval required for destructive,
identity-sensitive, security-sensitive, or potentially costly actions.

## Project status

Planning and team formation are complete. Implementation follows the approved
[design](docs/superpowers/specs/2026-08-02-oci-api-mcp-wrapper-design.md) and
[implementation plan](docs/superpowers/plans/2026-08-02-oci-api-mcp-wrapper.md).

No functional MCP server has been released yet.

## Security boundary

- The initial server will listen only on loopback.
- OCI credentials, private keys, raw profiles, runtime databases, and audit logs
  must never enter Git.
- Live OCI mutations require separate explicit approval and verified cleanup.
- Remote exposure, gateway routes, and firewall changes are outside the initial
  scope.

## Planning documents

- [Approved project summary](docs/a-team-summary.md)
- [Approved design specification](docs/superpowers/specs/2026-08-02-oci-api-mcp-wrapper-design.md)
- [Authoritative implementation plan](docs/superpowers/plans/2026-08-02-oci-api-mcp-wrapper.md)
- [Approved A-Team plan](docs/a-team-plan.md)
