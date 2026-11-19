# Injection Guard

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docs](https://img.shields.io/badge/docs-plugins-blue)](https://docs.mindroom.chat/plugins/)
[![Hooks](https://img.shields.io/badge/docs-hooks-blue)](https://docs.mindroom.chat/hooks/)

<picture>
  <source media="(prefers-reduced-motion: no-preference)" srcset="https://raw.githubusercontent.com/mindroom-ai/mindroom/main/assets/logo/logo-mark-animated.svg" />
  <img src="https://raw.githubusercontent.com/mindroom-ai/mindroom/main/assets/logo/logo-mark.svg" alt="MindRoom Logo" align="right" width="120" />
</picture>

A [MindRoom](https://github.com/mindroom-ai/mindroom) hook plugin that checks tool results carrying outside text, such as mail, web pages, search results, chat messages, and MCP tool output, for prompt injection before the model reads them.
A cheap checker model, chosen by the deployment, answers one strict question per result: does this text try to instruct an AI agent, for example to ignore its instructions, call tools, run commands, send data elsewhere, reveal secrets or its prompt, or change its task?
A clean result reaches the model unchanged; a flagged one is withheld or marked as untrusted data.

## How it works

1. A tool call returns a result, and MindRoom runs the `tool:result_transform` hooks on its text.
2. The hook skips an empty result and any tool that matches none of the `tools` globs, without a checker call.
3. A result longer than `max_chars` is cut to its first `max_chars` characters; only that part is checked, and the model reads it with one line saying how many characters were removed unchecked.
4. The kept text is split into chunks that each fit one judgment request (16,000 bytes with the rubric), overlapping by 200 characters so an instruction cut by a boundary is still seen whole.
   A chunk whose request grows too large through JSON escaping or multi-byte characters is split in half again.
   The middle of a long result is checked as fully as its start and end.
5. The chunks are checked concurrently, up to four at a time, with MindRoom's judgment backend; any chunk the checker flags makes the whole result a hit.
   Credentials and token-bearing URLs are redacted from the checker's copy, which needs no secrets; a clean result still reaches the model as it was.
6. On a hit, `withhold` replaces the result with a short notice naming the tool, without any of its text, and `mark` wraps the text between begin and end lines carrying a random per-call token, so the text cannot fake the end line.
7. Each check is logged through the plugin logger with the tool, agent, characters checked, chunk count, verdict (`clean`, `hit`, or `undecided`), and action; result text is never logged.

The hook fails closed.
When the checker fails or times out, has no backend (such as a missing credential), a chunk still cannot fit after splitting, or the settings are invalid, the hook raises, and MindRoom withholds the result: the model reads `The result of <tool> was withheld because a result check failed.`
A checker that answers undecided counts as no hit and is logged as `undecided`.
There is no retry.

There is no keyword pre-filter: patterns can only add hits and never clear text without the model, since they miss paraphrases, so they would save no checker calls on clean text.

## Hooks

| Hook | Event | Purpose |
|------|-------|---------|
| `injection_guard` | `tool:result_transform` | Check matching tool results for instructions aimed at an AI agent; withhold or mark hits |

## Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `judgment` | required | A MindRoom [judgment backend](https://docs.mindroom.chat/configuration/threads/#judgment-backends), such as `{provider: llm, model: <alias from models:>, timeout_seconds: 20}`. Required, so the deployment chooses and pays for the checker model. |
| `tools` | see below | `fnmatch` globs; a call matches when one matches its function name or its toolkit name. |
| `max_chars` | `50000` | Characters of a result that are checked and kept; at most `60000`, the redactor's input limit. |
| `action` | `withhold` | `withhold` replaces a hit with a notice; `mark` wraps it as untrusted data. |

```yaml
models:
  checker:
    provider: openai
    id: your-small-fast-model

plugins:
  - path: plugins/injection-guard-plugin
    settings:
      judgment:
        provider: llm
        model: checker
        timeout_seconds: 20
      action: withhold
      # tools: ["gmail", "website", "mcp_*", "my_fetch_*"]
    hooks:
      injection_guard:
        timeout_ms: 60000  # only for a slower checker or many-chunk results
```

Default `tools`, the built-in toolkits whose results carry outside text:

- mail and calendar: `gmail`, `outlook_mail`, `google_calendar`, `outlook_calendar`
- web fetch, scrape, and search: `website`, `browser`, `browser_mcp`, `web_browser_tools`, `browserbase`, `crawl4ai`, `firecrawl`, `jina`, `newspaper`, `trafilatura`, `spider`, `scrapegraph`, `agentql`, `brightdata`, `oxylabs`, `serper`, `serpapi`, `searxng`, `tavily`, `exa`, `linkup`, `duckduckgo`, `googlesearch`, `baidusearch`, `wikipedia`, `arxiv`, `pubmed`, `google_scholar`, `hackernews`, `reddit`, `youtube`, `x`
- chat networks the agent reads: `slack`, `discord`, `microsoft_teams`
- every MCP server: `mcp_*`

Setting `tools` replaces the whole list.
A function's toolkit comes from the `function_names` the built-in toolkits declare; a function name that several built-in toolkits declare, such as `send_message`, matches when any of them matches.
An MCP function, named `<prefix>_<remote tool>`, belongs to the toolkit `mcp_<server id>` of the server in `mcp_servers` whose `tool_prefix` (by default the server id) it starts with.
A plugin's own toolkit names are not known to the hook, so match a plugin's tools by their function names.

The default hook timeout is 30 seconds.
A 50,000-character result of plain text is about four chunks checked in parallel, which fits with `timeout_seconds: 20`; text with many multi-byte characters or escapes needs more chunks and runs four at a time, so raise `timeout_ms` under the plugin entry's `hooks:` as above for such results or for a slower checker.

## Install

Vendor this plugin with the MindRoom CLI:

```bash
mindroom plugins install injection-guard-plugin
```

Then reference it from `config.yaml` with its settings, as in the example above:

```yaml
plugins:
  - path: plugins/injection-guard-plugin
    settings:
      judgment: {provider: llm, model: checker, timeout_seconds: 20}
```

Update to the latest commit later with:

```bash
mindroom plugins update injection-guard-plugin
```

The command pins the exact installed commit in `.mindroom-plugin.lock.json` and strictly validates the plugin before activating it.
The plugin needs MindRoom with plugin API 1.32 (`tool:result_transform`); an older MindRoom skips it with a clear reason.

## Limits

- Images, audio, video, and files in a result are not checked; only its text is.
- Progress a tool streams while it runs is not checked.
- A delegated agent's answer to the agent that delegated to it is not checked; the delegated agent's own tool calls are.
- A checker can miss an attack or flag benign text, such as an article about prompt injection; `mark` keeps flagged text readable when false hits matter more than a stricter `withhold`.
- A result longer than `max_chars` is cut, and the model never reads the removed part.
- A checker error or timeout withholds the result. MindRoom's judgment backends run at most 8 requests at once across the process and refuse the rest, so many large guarded results at the same moment can withhold some of them.
- TODO-COST
