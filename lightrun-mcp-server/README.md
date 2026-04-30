# Lightrun MCP Server

Connect AI coding assistants to live runtime context from production and staging applications using Lightrun.

Lightrun MCP lets developers use MCP-compatible AI clients to safely inspect code-level runtime behavior without adding logs, redeploying, restarting, or interrupting the application. It helps AI assistants answer debugging questions using live data such as expression values, call stacks, execution duration, execution counts, and numeric runtime metrics.

## What is Lightrun MCP?

Lightrun MCP is a remote Model Context Protocol server exposed by Lightrun. It connects compatible AI assistants and agent frameworks to Lightrun's runtime debugging capabilities.

With Lightrun MCP, an assistant can help you:

- Discover available runtime sources, including agents, tags, and agent pools
- Inspect live expression values from running code
- Capture runtime call stacks
- Measure code execution duration
- Count how often a line of code executes
- Collect numeric runtime metrics and samples
- Debug production and staging behavior without changing code or redeploying

## MCP server URL

For Lightrun SaaS, use:

```json
{
  "mcpServers": {
    "Lightrun": {
      "url": "https://app.lightrun.com/mcp"
    }
  }
}
```

For single-tenant and self-hosted deployments, replace the URL with your organization's Lightrun MCP endpoint from the Lightrun MCP page in the Lightrun platform.

## MCP configuration filenames

MCP configuration file names and schemas vary by client.

- Cursor uses `.cursor/mcp.json` with a top-level `mcpServers` object.
- VS Code and GitHub Copilot Chat use `.vscode/mcp.json` with a top-level `servers` object.
- GitHub Copilot CLI uses `~/.copilot/mcp-config.json` with a top-level `mcpServers` object.
- Some MCP clients accept the server URL or JSON configuration directly in their UI.

Use the example that matches your client.

## Registry metadata

The repository root includes `server.json` metadata for MCP registry and marketplace submissions. The metadata points to this `lightrun-mcp-server` subfolder inside the `lightrun-ai` repository.

## Requirements

- A Lightrun account
- Access to a Lightrun organization
- A supported MCP-compatible client or agent framework
- Lightrun agents connected to the target runtime environment
- Appropriate Lightrun permissions for the runtime sources you want to inspect

## Authentication

Lightrun MCP supports authenticated access to your Lightrun account.

Typical authentication options include:

- OAuth for personal AI assistants
- API key Bearer authentication for headless agents, automation, and agent frameworks

Access is governed by Lightrun roles, permissions, API key scopes, and runtime source access controls.

## Example prompts

Use prompts like these from an MCP-compatible AI assistant:

```text
List the runtime sources I can debug with Lightrun.
```

```text
Inspect the value of userId and orderTotal at this line.
```

```text
Capture the call stack when this branch executes.
```

```text
Measure how long this method takes in production.
```

```text
Count how often this line executes over the next minute.
```

## Supported clients and frameworks

Lightrun MCP can be used with MCP-compatible clients and frameworks, including:

- Cursor
- GitHub Copilot
- Claude Code
- Amazon Q
- Gemini Code Assist
- JetBrains AI Assistant
- OpenAI Agents
- LangChain
- LlamaIndex
- n8n

See the official quickstart for client-specific setup instructions:

https://docs.lightrun.com/mcp/mcp-quickstart/

## Documentation

- Quickstart: https://docs.lightrun.com/mcp/mcp-quickstart/
- MCP overview: https://docs.lightrun.com/mcp/mcp-overview/
- Supported Lightrun MCP tools: https://docs.lightrun.com/mcp/lightrun-tools/

## Security

Lightrun MCP uses Lightrun's existing authentication, authorization, and runtime access controls. MCP clients only operate within the permissions granted to the authenticated user or API key.

For responsible disclosure or security concerns, contact support@lightrun.com.

## Support

For help, contact support@lightrun.com or visit the Lightrun documentation:

https://docs.lightrun.com/

## License

This repository contains documentation and sample configuration files for connecting to Lightrun MCP. See [LICENSE](LICENSE).
