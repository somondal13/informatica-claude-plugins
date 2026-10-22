## Security

Please report any security issue to [https://www.sfdc.co/SubmitVuln](https://www.sfdc.co/SubmitVuln)
as soon as it is discovered. This is Salesforce's standard responsible-disclosure
intake and applies across Salesforce open source projects, including this one.

This repo has no runtime dependencies — it is a Claude Code plugin
container consisting of JSON manifests and Markdown skill definitions.
The main security-relevant surface is MCP server configuration (URLs,
credentials, auth) inside each plugin — review any such change with
extra care, and flag it through the link above if you believe it
introduces a vulnerability.
