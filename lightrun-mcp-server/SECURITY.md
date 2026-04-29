# Security Policy

## Reporting a vulnerability

If you believe you have found a security vulnerability related to Lightrun MCP, please contact:

support@lightrun.com

Please include:

- A clear description of the issue
- Steps to reproduce, if applicable
- The affected Lightrun environment or MCP client configuration
- Any relevant logs, screenshots, or request examples

## Authentication and authorization

Lightrun MCP requires authenticated access to a Lightrun account. Access is governed by Lightrun roles, permissions, API key scopes, and runtime source access controls.

Do not publish API keys, OAuth tokens, bearer tokens, customer identifiers, or internal environment URLs in GitHub issues, pull requests, screenshots, or examples.

## Sensitive data guidance

When using Lightrun MCP:

- Avoid inspecting secrets, credentials, tokens, or personally identifiable information unless explicitly authorized.
- Scope runtime inspections to the minimum code area needed for debugging.
- Follow your organization's production debugging and data handling policies.
