# MCP Servers for Business Central AL Development

A practical guide to choosing and using MCP servers for Microsoft Dynamics 365 Business Central AL development. It covers a curated set of 15 servers, with installation, configuration, verification, and maintenance guidance focused on Codex on native Windows. The tool-selection principles also apply to other agents, though their configuration and approval flows differ.

## Start here

1. Read the [MCP server guide](Best-MCP-Servers-for-BC-Development.md) and choose only the servers your task needs.
2. Follow the guide's [installation instructions](Best-MCP-Servers-for-BC-Development.md#per-server-installation) and [configuration reference](Best-MCP-Servers-for-BC-Development.md#configuration-reference).
3. Run the [smoke checks](Best-MCP-Servers-for-BC-Development.md#verification) before relying on a server. Check the target project, permissions, and tool version before any write or publication.

## What's inside

| Resource | Use it for |
| --- | --- |
| [MCP server guide](Best-MCP-Servers-for-BC-Development.md) | The ranked toolbox, per-server setup, paste-ready Codex configuration, troubleshooting, maintenance, and a shared AL agent policy. |
| [RDL layout skill](skills/al-rdl-layout/SKILL.md) | Inspecting and making supported, narrowly scoped edits to existing BC RDL/RDLC layouts with `rdl-mcp`. |
| [RDL layout design skill](skills/al-rdl-layout-design/SKILL.md) | Deciding whether RDL is appropriate and planning trading documents, list reports, and statements before editing a layout. |

The guide covers AL compilation and symbols, Business Central source investigation, documentation, report layouts, browser checks, work tracking, and approved tenant or administration tasks. It distinguishes tools that are generally useful from those best enabled only for a specific task.

## Boundaries

- Prefer local evidence and the smallest relevant toolset; an available tool is not authorization to use it.
- Confirm versions and advertised tool schemas against your environment. The guide records what was tested, not permanent version requirements.
- For reports, an XML check or successful AL build does not prove the rendered output. Verify representative output in a Business Central sandbox before sign-off.
