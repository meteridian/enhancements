# ADR-0008: AI-First Interface (MCP + A2A) over Visual Editor

- **Status:** Accepted
- **Date:** 2026-06-18
- **Deciders:** @pgarciaq, @jordigilh

## Context

Users need to create, configure, monitor, and optimize Meteridian processing
pipelines. These pipelines are DAGs of typed processing blocks (see ADR-0007)
with complex configuration surfaces: Arrow schemas, capability declarations,
connection topology, performance tuning, and error handling policies.

The traditional approach for dataflow systems is a visual editor -- Node-RED,
Apache NiFi, and AWS Step Functions all provide drag-and-drop canvas UIs. These
are intuitive for simple pipelines but become unwieldy for complex configurations
and impossible to use programmatically (CI/CD, infrastructure-as-code).

Meanwhile, the AI agent ecosystem has matured rapidly. The Model Context Protocol
(MCP) provides a standard way for AI assistants to discover and invoke tools.
The Agent-to-Agent (A2A) protocol enables autonomous agent collaboration. These
protocols let any AI assistant -- Claude, ChatGPT, Codex, Kimi, Gemini -- interact
with Meteridian natively, without bespoke integration per platform.

The Kubernaut project (github.com/jordigilh/kubernaut) demonstrates a production
implementation of this pattern: a unified `apifrontend` service exposing both MCP
and A2A through a centralized tool registry, with agent cards auto-generated from
the MCP tool definitions.

## Decision

Meteridian adopts an **AI-first extensibility model**: the Model Context Protocol
(MCP) server and Agent-to-Agent (A2A) agent are the **primary** programmatic
interfaces for pipeline management. A visual editor is a secondary, read-mostly
UI that renders pipeline state but does not drive the interaction model.

### MCP Server

Meteridian exposes an MCP-compliant tool server with tools including:

- `list_blocks` -- Discover available block types and their schemas
- `create_block` -- Instantiate a new block from a type + configuration
- `create_pipeline` -- Define a complete pipeline from a block topology
- `get_block_metrics` -- Retrieve per-block latency, throughput, error rates
- `optimize_pipeline` -- Suggest or apply performance improvements
- `debug_block` -- Inspect block state, recent errors, and input/output samples
- `search_marketplace` -- Find blocks by capability, category, or description
- `install_block` -- Install a marketplace block into the tenant's environment

Tools are registered in a centralized `mcpToolRegistry` (following the Kubernaut
pattern from `pkg/apifrontend/handler/mcptools.go`), ensuring a single source of
truth for tool definitions shared between MCP and A2A.

### A2A Agent

Meteridian publishes an agent card at `/.well-known/agent-card.json` with skills
auto-derived from the MCP tool registry (the `DefaultAgentSkills` pattern from
Kubernaut's `agentcard.go`). This enables autonomous agent-to-agent interactions:

- An external optimization agent requests pipeline metrics, analyzes bottlenecks,
  and proposes configuration changes
- A compliance agent audits block provenance and capability declarations
- A cost forecasting agent analyzes usage patterns and recommends block changes

### Visual Editor (Secondary)

A Node-RED-inspired web UI for pipeline visualization. In v1, it is read-only
(renders pipeline YAML as a visual graph). In v2+, it gains editing capabilities.
The visual editor is auto-generated from the same pipeline definition that MCP
and A2A operate on -- it is a view, not the source of truth.

### Implementation Reference

The Kubernaut `apifrontend` package provides a proven blueprint:

- **`router.go`**: HTTP router with health, metrics, agent card, and authenticated
  MCP and A2A endpoints, applying middleware (auth, CORS, audit logging)
- **`mcp.go`**: `NewMCPHandler` using `github.com/modelcontextprotocol/go-sdk/mcp`
  for the MCP Streamable HTTP protocol
- **`agentcard.go`**: A2A agent card using `github.com/a2aproject/a2a-go/a2a`,
  with `DefaultAgentSkills` deriving A2A skills from the MCP tool registry
- **`mcptools.go`**: `mcpToolRegistry` as the centralized tool definition list

Meteridian will adapt this pattern, replacing Kubernaut's Kubernetes-focused tools
with metering and billing-focused tools.

## Consequences

### Positive

- **Universal AI access**: Any MCP-compatible AI assistant can manage Meteridian
  pipelines without custom integration code
- **Automation-native**: Pipelines can be created, tested, and deployed entirely
  via AI agents in CI/CD workflows
- **Discoverability**: AI agents can explore available blocks, read documentation,
  and suggest configurations -- a better UX than browsing a visual catalog
- **Lower development cost**: MCP server + A2A agent is ~2 months of work vs
  6-12 months for a polished visual editor
- **Composability**: External agents can orchestrate multiple Meteridian instances,
  compare configurations across environments, and implement cross-platform workflows

### Negative

- **AI dependency**: Users without AI assistant access must use YAML or CLI directly
  (adequate for developers, but less accessible for business users)
- **Protocol maturity**: MCP and A2A are young protocols (2024-2025); breaking
  changes are possible
- **Testing complexity**: AI-driven interactions are harder to test deterministically
  than UI interactions

### Neutral

- The visual editor is still planned (v2+) -- this decision deprioritizes it,
  not eliminates it
- YAML pipeline definitions remain the source of truth regardless of which
  interface creates them

## Alternatives Considered

### Visual Editor First (Node-RED and NiFi Style)

A drag-and-drop canvas UI for pipeline design. Intuitive for simple pipelines and
non-technical users.

**Rejected because:**
- 6-12 month development effort for a production-quality editor (React Flow,
  state management, undo/redo, validation, real-time preview)
- Visual editors struggle with complex configurations (30+ blocks, conditional
  routing, dynamic schemas)
- Not automation-friendly -- CI/CD pipelines cannot interact with a visual canvas
- Every AI assistant would need a custom browser automation layer to use it
- The visual representation can be auto-generated from YAML later; the reverse
  (generating clean YAML from a visual canvas) is much harder

### CLI-Only Interface

A powerful CLI tool (`metr pipeline create`, `metr block install`, etc.) with
shell completion and rich output formatting.

**Rejected as primary because:**
- Steep learning curve for new users
- No discoverability (must read docs to know what's possible)
- Not AI-native -- AI agents can invoke CLI commands but it's awkward compared
  to structured MCP tool calls with typed parameters and responses
- Still planned as a complementary interface alongside MCP

### REST API Only

Standard REST endpoints for all pipeline operations.

**Rejected as primary because:**
- Every AI platform would need custom HTTP integration code
- No standard discovery mechanism (OpenAPI helps but isn't interactive)
- MCP provides a higher-level abstraction that wraps REST internally while
  offering a standardized tool interface to AI agents

## References

- Model Context Protocol: https://modelcontextprotocol.io/
- A2A Protocol: https://github.com/a2aproject/a2a-go
- Kubernaut (reference implementation): https://github.com/jordigilh/kubernaut
- Kubernaut apifrontend: https://github.com/jordigilh/kubernaut/tree/main/pkg/apifrontend
- Go MCP SDK: https://github.com/modelcontextprotocol/go-sdk
- Node-RED: https://nodered.org/
- React Flow: https://reactflow.dev/
