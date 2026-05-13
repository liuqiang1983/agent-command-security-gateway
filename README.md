# Agent Command Security Gateway

## Version

MVP Detailed Design Frozen v0.5.1-rc.1

## Purpose

A security gateway for AI Agent command execution that enforces zero-trust principles and provides comprehensive audit logging.

## Core Principles

1. **All commands MUST go through Command Gateway**
2. **Executor MUST NOT be called directly**
3. **Zero-trust fallback for all uncertain cases**
4. **All security decisions MUST be audited**

## Architecture

```
Agent Request
  -> CommandExecutionGateway
  -> Tokenizer
  -> Normalizer
  -> Classifier
  -> DependsResolver
  -> PolicyEngine
  -> Executor (only if ALLOW)
```

## Security Invariants

- A1: Gateway Mandatory - All commands must go through Gateway
- A2: Zero Trust Fallback - Uncertainty must not auto-allow
- A3: Depends Resolution Completeness - No unresolved depends before Policy Engine
- A4: Cache Fast-Path Forbidden Cases - 10 specific forbidden cases
- A5: UserDeniedSimilar Semantics - Based on normalized token array prefix match
- A6: Event Auditability - All security decisions must be auditable

## Development

```bash
pnpm install
pnpm build
pnpm test
```

## License

MIT

## Security Behavior

### Critical Risk Commands

By default, commands classified as **critical risk** (e.g., \`rm -rf /\`, \`mkfs\`, \`dd\` to disk) will trigger an **ask** decision, requiring explicit user approval. This is controlled by the \`denyCriticalRisk\` policy option (default: \`false\`).

To automatically deny critical risk commands without asking:
\`\`\`typescript
const gateway = new CommandExecutionGateway({
  policyConfig: { denyCriticalRisk: true }
});
\`\`\`

### Tokenizer Security Restrictions

The tokenizer enforces strict security rules:

1. **Semicolons (\`;\`) are always rejected** - Including escaped forms like \`\;\`. This prevents command chaining attacks.
   - \`echo hello; rm -rf /\` → **REJECTED**
   - \`find . -exec cmd \;\` → **REJECTED** (use \`find . -exec cmd +\` instead)

2. **Dangerous patterns are blocked** - Backticks, \`$()\`, \`&&\`, \`||\`, pipes, and redirections are forbidden.

3. **Uncertain input triggers ask** - Any tokenization ambiguity results in a security prompt.

### UserDeniedSimilar Isolation

The \`UserDeniedSimilar\` cache is isolated by \`toolName\`. A denial in one tool does not affect other tools:
- Tool A denies \`rm -rf /tmp\` → Tool B can still ask about \`rm -rf /tmp\`
- This prevents cross-tool denial propagation.

### Audit Events

All security decisions generate audit events:
- \`GatewayAllowed\` - Command approved and executed
- \`GatewayDenied\` - Command denied by policy
- \`GatewayDeniedSimilar\` - Command denied due to similar previously denied command
- \`GatewayCacheHit\` - Command matched cache (allowed or denied)
- \`GatewayCommandDenied\` - Command denied by risk classification
- \`GatewayExecutionFailed\` - Command execution failed

### Approval Request Identity

When the Gateway requests user approval, the approval request includes complete command identity:

- `rawCommand` - The original command string as received
- `argv` - Tokenized argument array (non-empty, validated by tokenizer)
- `auditId` - Unique identifier for audit traceability
- `context` - Session, user, workspace, and agent identifiers where available

**Important**: Approval responses cannot replace or modify the command being executed. The approved command is executed exactly as requested.

### Known Audit Testability Limitations

The following audit event branches exist in the code but are not directly testable through the current public Gateway API:

1. **GatewayCommandDenied** - This event is emitted when a command is denied by risk classification. Testing requires `policyConfig` injection to set `denyCriticalRisk: true`, which is not exposed through the public Gateway constructor.

2. **GatewayExecutionFailed** - This event is emitted when command execution fails. Testing requires `executor` injection with a failing executor, which is not exposed through the public Gateway constructor.

These branches are covered by code review and static analysis, but not by automated integration tests.
