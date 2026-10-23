# Splunk MCP dashboard tools

Two custom tools for the Splunk MCP Server app that let an AI assistant create and edit dashboards, locked to one dedicated Splunk app.

| Tool | What it does | Splunk REST call |
|------|--------------|------------------|
| `splunk_create_dashboard` | Creates a new dashboard | `POST /servicesNS/nobody/custom_app/data/ui/views` |
| `splunk_update_dashboard` | Replaces the source of an existing dashboard | `POST /servicesNS/nobody/custom_app/data/ui/views/<dashboard_name>` |

Both tools take a dashboard ID and the full dashboard source, either Simple XML (`version="1.1"`) or Dashboard Studio (`version="2"`).

## Why one dedicated app

Both tools are locked to a single app, `custom_app`, on purpose. The app name is written into each tool's endpoint instead of being an argument, so the assistant can't pick another app. Everything it creates lands in `custom_app` and every update goes through it. That keeps the AI's write access to a small, known area, and everything it builds sits in one place where you can review it or clean it up.

To keep that boundary tight:

- Anything another app shares with all apps (global sharing) is also visible inside `custom_app`, including some of Splunk's own pages, so the update tool can reach it by name.
- For a hard limit, connect the MCP client as a Splunk user whose roles can write to `custom_app` but not to other apps.
- You can also put a prefix in the `dashboard_name` pattern of both tools, for example `^ai_[A-Za-z0-9_-]+$`, so the assistant can only create and update dashboards whose ID starts with `ai_`.

## Files

- [`tools/splunk_create_dashboard.json`](tools/splunk_create_dashboard.json)
- [`tools/splunk_update_dashboard.json`](tools/splunk_update_dashboard.json)

## Setup

1. Create a dedicated app for the dashboards the assistant builds (**Apps → Manage Apps → Create app**). The definitions use the folder name `custom_app`; if you choose a different one, change it in the `endpoint` and `description` of both files.
2. In the Splunk MCP Server app, go to **Tools → Create Tool** and enter each definition with execution type **API**. You can also register them through the app's REST endpoint, `/services/mcp_tools` ([Splunk docs](https://help.splunk.com/en/splunk-cloud-platform/mcp-server-for-splunk-platform/2.0/managing-custom-tools-in-splunk-mcp-server)).
3. Enable both tools, then reconnect your MCP client so it picks them up.

The Splunk user your MCP client connects as needs write access to `custom_app`.

## How they behave

- Placeholders use the MCP Server's `$param$` syntax, not `{param}`.
- New dashboards are shared at app level and owned by the user the MCP client connects as.
- An update replaces the whole source. The update tool's description tells the assistant to read the current source first:

  ```
  | rest /servicesNS/nobody/custom_app/data/ui/views/<dashboard_name> splunk_server=local | fields eai:data
  ```

- The update tool can't reach private dashboards. Share a dashboard with the app before asking the assistant to edit it.
- `dashboard_name` must match `^[A-Za-z0-9_-]+$`. The update tool puts it in the URL, so keep that pattern.
- Dashboard tokens such as `$tok$`, `$time.earliest$` and the `$$` escape pass through unchanged.

## Tested

On Splunk Enterprise 10.4.1 with Splunk MCP Server 2.0.0, using Simple XML dashboards:

- Create puts the dashboard in the target app, shared with the app.
- Update changes the dashboard in place, with no duplicate copy in any app.
- Tokens in the source come through unchanged.
- Updating a dashboard that doesn't exist fails with `Could not find object` and creates nothing.

## License

[MIT](../../LICENSE)
