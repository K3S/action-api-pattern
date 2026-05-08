---
title: Calling Actions from MCP
nav_order: 8
permalink: /calling-from-mcp/
---

# Calling Actions from MCP
{: .no_toc }

1. TOC
{:toc}

---

This is where the pattern pays off in a way that wasn't obvious until 2024 or so. An Action API is, almost exactly, the right shape for an MCP tool. The work to expose your IBM i to AI agents is mostly translation, not redesign.

## What MCP is, briefly

The Model Context Protocol (MCP) is an open specification that defines how an AI agent — typically an LLM-based assistant — talks to external systems. An MCP server hosts a set of *tools*. Each tool has a name, a description, an input schema (in JSON Schema), and an implementation. The agent reads the tool list, decides which tool to call based on what the user asked, formats arguments according to the schema, and gets a structured response back.

That's the entire shape of the protocol. An MCP tool is, fundamentally, *a single business action with a typed input and a typed output, callable by name*. Which is what an Action API already is.

So the bridge work is almost entirely mechanical: each Action API becomes one MCP tool. The CL wrapper becomes the dispatch layer. The error envelope becomes the tool's success/failure response. The `ID*`/`SB*`/`RV*` parameters become the input and output schemas.

## Architecture

The MCP server is a separate process. It can run anywhere — on the IBM i itself (Node.js or PHP under PASE), on a Linux server adjacent to the IBM i, or in a container in a Kubernetes cluster. It exposes the MCP transport (stdio for desktop clients, HTTP/SSE for networked clients), and it talks to the IBM i through the same toolkit you used in the previous chapter.

```
┌──────────────────┐        MCP        ┌──────────────────┐
│   AI Agent       │  ───────────────▶ │   MCP Server     │
│   (Claude, etc.) │                   │   (Node or PHP)  │
└──────────────────┘                   └─────────┬────────┘
                                                 │ Toolkit / XMLSERVICE
                                                 ▼
                                       ┌──────────────────┐
                                       │   IBM i          │
                                       │   AC_DELSUPL     │
                                       │   AR_DELSUPL     │
                                       └──────────────────┘
```

We'll show a TypeScript implementation here because the MCP TypeScript SDK is the most mature, but the same structure applies to any language with an MCP SDK.

## Defining a tool that calls an Action API

Each tool is one Action. Here's `delete_supplier`:

```typescript
import { z } from "zod";
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { CallToolRequestSchema, ListToolsRequestSchema }
  from "@modelcontextprotocol/sdk/types.js";
import { callIbmiAction } from "./ibmi-gateway.js";

const server = new Server(
  { name: "k3s-action-api", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// ---------------------------------------------------------------
// Tool definitions — one per Action API
// ---------------------------------------------------------------
const tools = [
  {
    name: "delete_supplier",
    description: [
      "Delete a supplier from the K3S supplier file. The supplier",
      "is identified by a compound key: buyer code, location code,",
      "supplier code, and optional supplier sub-code. Will fail if",
      "the supplier has open purchase orders.",
    ].join(" "),
    inputSchema: {
      type: "object",
      properties: {
        buyer_code: {
          type: "string",
          description: "5-character buyer code (IDBUYR)",
          maxLength: 5,
        },
        location_code: {
          type: "string",
          description: "5-character location code (IDLOCN)",
          maxLength: 5,
        },
        supplier_code: {
          type: "string",
          description: "10-character supplier code (IDSUPL)",
          maxLength: 10,
        },
        supplier_sub: {
          type: "string",
          description: "Optional supplier sub-qualifier (IDSUPLSUB)",
          maxLength: 10,
          default: "",
        },
      },
      required: ["buyer_code", "location_code", "supplier_code"],
    },
  },
  // ... other tools (add_supplier, update_supplier, etc.) ...
];

// ---------------------------------------------------------------
// List tools handler — agents call this once to discover capabilities
// ---------------------------------------------------------------
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools,
}));

// ---------------------------------------------------------------
// Call tool handler — dispatches to the right Action API
// ---------------------------------------------------------------
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "delete_supplier":
      return await callIbmiAction("AC_DELSUPL", [
        { name: "IDBUYR",    size: 5,  value: args.buyer_code },
        { name: "IDLOCN",    size: 5,  value: args.location_code },
        { name: "IDSUPL",    size: 10, value: args.supplier_code },
        { name: "IDSUPLSUB", size: 10, value: args.supplier_sub ?? "" },
      ]);

    case "get_supplier":
      // ... and so on
      break;

    default:
      throw new Error(`Unknown tool: ${name}`);
  }
});
```

The `callIbmiAction` function is the gateway — it builds the standard envelope (`K3SOBJ`, `COMP`, `COMPCOD`, `USER`, `ERRORS`, `ERRMSG`, `ERRFIELD`), prepends it to the action-specific parameters, dispatches the call to the IBM i (typically by HTTP POSTing to a thin PHP endpoint, or by direct ODBC to call the program), and returns the result in MCP's expected format:

```typescript
async function callIbmiAction(programName: string, actionParams: Param[]) {
  const envelopeParams = buildEnvelope();   // K3SOBJ, COMP, etc.
  const allParams = [...envelopeParams, ...actionParams];

  const result = await ibmiGateway.call(programName, allParams);

  if (result.errors === "Y") {
    return {
      content: [{
        type: "text",
        text: `Action failed: ${result.errmsg} (field: ${result.errfield})`,
      }],
      isError: true,
    };
  }

  return {
    content: [{
      type: "text",
      text: `Action completed successfully.`,
    }],
  };
}
```

That's it. The MCP server hosts one tool per Action API, dispatches each call into the IBM i, and translates the response envelope into MCP's content format.

## Why the schema mapping is so clean

Look at what just happened. The `inputSchema` for `delete_supplier` is essentially the parameter list of `AR_DELSUPL` rendered as JSON Schema. The `ID*` fields became `required`. The `SB*` fields, if there were any, would be required if the Action requires them and optional otherwise. The lengths from the RPG declarations became `maxLength` constraints.

You could literally generate the MCP tool definitions from the Action API documentation. K3S maintains a YAML manifest of Action APIs that drives both the technical docs site and the MCP tool registration. One source of truth, two consumers. The MCP server reads the manifest at startup and registers tools automatically; adding a new Action means adding an entry to the manifest, no code change in the server.

A simplified manifest entry:

```yaml
- name: delete_supplier
  program: AC_DELSUPL
  description: Delete a supplier from the K3S supplier file...
  parameters:
    - name: IDBUYR
      schema_name: buyer_code
      type: char
      size: 5
      required: true
      description: 5-character buyer code
    - name: IDLOCN
      schema_name: location_code
      type: char
      size: 5
      required: true
      description: 5-character location code
    - name: IDSUPL
      schema_name: supplier_code
      type: char
      size: 10
      required: true
      description: 10-character supplier code
    - name: IDSUPLSUB
      schema_name: supplier_sub
      type: char
      size: 10
      required: false
      default: ""
      description: Optional supplier sub-qualifier
```

The MCP server iterates the manifest, builds tool definitions, and registers handlers. When a new Action is added to the manifest, restart the server and the agent has a new tool.

This mechanical mapping is the hidden gift of the Action API Pattern. We didn't design it for MCP — MCP didn't exist when we started. But the same constraints that made the pattern good for PHP (single purpose, typed parameters, structured response) make it perfect for AI tools. Strong abstractions transfer across protocols you didn't know about yet.

## Authorization and scoping

There's a problem to take seriously: an AI agent calling `delete_supplier` could, in principle, delete any supplier in the system. That's not a flaw of MCP; it's a property of giving any caller direct access to a destructive Action.

The mitigations are the same ones you'd apply to any privileged API:

**Per-tool authorization.** The MCP server runs as a particular user. That user can be configured with limited authority on the IBM i — e.g., read access to most files but no delete authority on `K_SUPPL`. Even if the agent calls `delete_supplier`, the IBM i refuses. The Action-level permission model is the safety net.

**Tool-level allow-listing.** The MCP server doesn't have to expose every Action. For an agent serving a customer-service rep, expose `get_supplier`, `list_suppliers`, `update_supplier_phone`, but *not* `delete_supplier`. For a buyer's agent, expose more. Different MCP server configurations for different audiences.

**Confirmation flows.** Destructive Actions can be wrapped in a two-step pattern: the MCP tool returns "this will delete supplier ACME01 with X open relationships, confirm?" and waits for a separate "confirm_deletion" tool call. This adds friction but eliminates entire classes of mistakes. Whether to require it depends on the deployment.

**Audit logging.** Every Action call is already logged through the API log. The agent's calls are no exception. If something deletes a supplier it shouldn't have, the log shows what asked for it, when, with what parameters. Combine this with the MCP server's own logging (the request from the agent, the user the agent was acting on behalf of) and you have full traceability.

{: .warning }
> Don't expose destructive Actions to AI agents in your first MCP deployment. Start read-only — `get_supplier`, `list_suppliers`, `count_open_orders`, `report_status`. Build trust with the read-only surface, watch how agents actually use the tools, then introduce mutations carefully and with confirmation flows. The Action API Pattern doesn't change the importance of judgment about what the agent should be allowed to do.

## The K3S MCP server in practice

The K3S MCP server, at the time of writing, exposes about 40 tools — a mix of supplier, product, order, and reporting actions. It's a Node.js process running on a Linux box adjacent to the IBM i, talking to the toolkit through a thin PHP gateway over HTTP. The setup runs in Docker and pulls the manifest from the same repository as the technical docs.

When a customer's user asks an AI assistant *"delete supplier ACME01 from buyer 00001"*, the agent finds the `delete_supplier` tool, formats the arguments, and calls it. The MCP server builds the envelope, calls `AC_DELSUPL`, the CL calls `AR_DELSUPL`, the validation runs, the supplier deletes (or doesn't), the envelope comes back. The agent reports success or quotes the error message.

End to end latency is ~150ms in production. Most of that is the agent's own thinking and the MCP transport; the IBM i portion is 30–80ms, the same as the web frontend.

## What we didn't have to change

This is the punchline. To expose the entire supplier API family to AI agents, we changed:

- **The MCP server.** New process, new code (~500 lines of TypeScript), new deployment.
- **The manifest.** Existed for docs already; added a few schema-mapping fields.

We did not change:

- The RPG. Not one line.
- The CL wrappers. Not one line.
- The TS_ test programs. Not one line.
- The PHP handlers. Not one line.
- The database. Not one line.

The Action API Pattern was already the right abstraction. MCP is a new client; the existing contract supports it natively. That's what good architecture looks like — adding a new client mode is a feature flag, not a project.

[Next: What You Don't Do]({% link 08-discipline.md %}){: .btn .btn-primary }
