# Client MCP compatibility for the standalone pilot

Research completed 2026-09-12 against fetched official documentation. This is a documentation baseline, not an account-level connection test.

## User requirements

The pilot supports REST and MCP with the same general database operations. One reusable connector per client accepts a database link identifying shared data. The first workflow is the user's shopping list across sessions. Target clients are ChatGPT Work, Grok website/mobile, and the new Grok Bot. The ChatGPT plugin starts with personal use; public distribution follows later. See the [pilot contract discussion](https://github.com/0000-chat/0000-database/issues/4).

## Findings

### ChatGPT Work and the ChatGPT plugin

OpenAI currently documents plugins that package skills, MCP servers, and optional UI. An MCP server can return structured results without a custom UI. This is a current plugin route, not a recommendation to revive an older integration format. [Plugin architecture](https://developers.openai.com/plugins/concepts/plugins)

The documented development path is to test the MCP server in developer mode, then package and evaluate the plugin. Public endpoints use HTTPS and Streamable HTTP, typically at `/mcp`. Developer-mode availability depends on the account and workspace policy. Public distribution is a separate submission step and need not block personal testing. Actual availability in the user's ChatGPT Work account has not been inspected. [Connect and test](https://developers.openai.com/plugins/deploy/connect-chatgpt)

OpenAI's authentication guidance recommends authentication for customer-specific data and write actions, and OAuth 2.1 for authenticated servers. Therefore an accountless, shared-link write service must not be described as proven compatible or publication-ready. The documentation does not establish whether this user's developer-mode connection will accept our precise capability-link write design. Verify it experimentally before fixing that part of the contract; do not silently introduce accounts or OAuth as a decided requirement. [Authentication](https://developers.openai.com/plugins/build/auth)

### Grok website

The consumer Grok documentation explicitly allows custom MCP connectors through `grok.com/connectors`, using New Connector, Custom, and a server URL. Tools are discovered for use in conversations. The server must be reachable publicly. This is evidence for the Grok product itself, not an inference from xAI API support. [Grok connectors](https://docs.x.ai/grok/connectors)

The tunneling guide discusses Streamable HTTP, and authentication when required by the server. This supports proposing one public HTTPS Streamable HTTP endpoint for both ChatGPT and Grok. It does not verify our exact tool schemas, anonymous write behavior, or the user's account. [Custom MCP tunneling](https://docs.x.ai/grok/connectors/custom-mcp-tunneling)

For Business and Enterprise accounts, admins must provision connectors before members use them. Whether that applies to the user's account is unknown. [Connector management](https://docs.x.ai/grok/connector-management)

### Grok mobile app

The fetched consumer connector guide gives a web setup path. It does not explicitly establish custom-MCP parity for the ordinary Grok iOS and Android apps. Keep mobile invocation as a separate account/device test. Do not transfer evidence about Grok Bot mobile to the ordinary Grok mobile app. [Grok connectors](https://docs.x.ai/grok/connectors)

### Grok Bot

Grok Bot is the persistent-agent product announced August 11, 2026, with a cloud computer and continuing work across sessions. It is distinct from the Grok social account and the xAI API. [Announcement](https://x.ai/news/introducing-grok-bot)

The Bot overview describes connectors and MCP where available. Its connection guide documents Settings, Plugins, choosing an available connector, authentication if requested, and attaching the connector in chat. Installed connectors are account-wide. Those pages do not establish an arbitrary custom-MCP URL installation flow for a personal Bot account. General MCP support alone is insufficient to claim our service is installable. [Bot overview](https://docs.x.ai/grok-bot/overview), [Computer and apps](https://docs.x.ai/grok-bot/computer-and-apps)

Bot mobile explicitly uses the same Bots, connectors, conversations, and cloud computer as desktop; its settings can install or review plugins. This establishes shared installed-connector behavior, but does not resolve the arbitrary-server installation question. Eligibility and settings can vary by account and rollout; inspect the actual account rather than promising access from a launch announcement. [Bot mobile](https://docs.x.ai/grok-bot/mobile), [Settings](https://docs.x.ai/grok-bot/settings-and-notifications)

## Recommended contract direction

These are engineering recommendations derived from the sources and the user decisions, not additional user-approved product policy:

- Expose one hosted HTTPS Streamable HTTP MCP endpoint alongside REST, using shared operations and validation.
- Pass the database identifier/link as operation input, preserving one reusable connector and the accepted possession-based access model.
- Package ChatGPT workflow instructions with the MCP connection after testing the personal connection. Keep custom UI and public distribution outside the first required integration milestone.
- Test the same endpoint on Grok web; independently verify ordinary Grok mobile and custom-server installation in Grok Bot.
- Do not make a blanket claim that all requested clients already work, and do not replace the requested Grok products with the xAI API.

## Remaining empirical checks

Documentation research is complete enough to choose a transport direction. Compatibility still needs a small executable probe once a remote endpoint exists:

1. In the user's ChatGPT Work account, discover tools and perform an explicit write and read using a throwaway database link. Determine whether no-account writes work or an authentication policy prevents them.
2. Repeat on Grok web and ordinary Grok mobile, retaining the same database identity across sessions.
3. Check Grok Bot's actual personal plugin installation options. If custom MCP installation is available, repeat the write/read and mobile tests. Otherwise record the platform limitation and return the release-scope decision to the user.

Do not change the accountless access policy or drop a target client without that decision. These tests can be pursued alongside durable storage work once the relevant contract and storage choices are settled.

## Validation limits

Official pages were searched and fetched; no client was logged into and no live database endpoint was tested. No product implementation changed. Repository documentation checks validate this artifact's repository compliance, not cross-client compatibility.
