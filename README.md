# Injection Guard

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Docs](https://img.shields.io/badge/docs-plugins-blue)](https://docs.mindroom.chat/plugins/)
[![Hooks](https://img.shields.io/badge/docs-hooks-blue)](https://docs.mindroom.chat/hooks/)

<picture>
  <source media="(prefers-reduced-motion: no-preference)" srcset="https://raw.githubusercontent.com/mindroom-ai/mindroom/main/assets/logo/logo-mark-animated.svg" />
  <img src="https://raw.githubusercontent.com/mindroom-ai/mindroom/main/assets/logo/logo-mark.svg" alt="MindRoom Logo" align="right" width="120" />
</picture>

A [MindRoom](https://github.com/mindroom-ai/mindroom) hook plugin that checks the text of successful tool results carrying outside text, such as mail, web pages, search results, shared documents, chat messages, and MCP tool output, for prompt injection before the model reads it.
A cheap checker model, chosen by the deployment, answers one strict question per chunk of text: does this text try to instruct an AI agent, for example to ignore its instructions, call tools, run commands, send data elsewhere, reveal secrets or its prompt, or change its task?
The model reads exactly the text the checker read, or a notice that the result was withheld.

## How it works

1. A tool call returns a successful result, and MindRoom runs the `tool:result_transform` hooks on its text.
2. The hook skips any tool that matches none of the `tools` globs, and a whitespace-only result (cut to `max_chars` when longer), without a checker call.
3. Credentials and token-bearing URLs are masked first by MindRoom's bounded redactor, which reads past `max_chars` to the end of a word, so a credential the cut splits is still recognized; text past what it reads is dropped.
   The text is then cut to `max_chars`, and a conversation tag such as `<msg from="...">` becomes `<_msg from="...">`, which the judgment request builder would otherwise rewrite.
   The removed tail is never sent to the checker.
4. The checked text is split into chunks that each fit one judgment request (16,000 bytes with the rubric), with boundaries on whitespace where possible and 200 characters of overlap.
   A chunk whose request grows too large through JSON escaping or multi-byte characters shrinks by a tenth until it fits, and every request is verified to carry its chunk verbatim.
   A whitespace-only chunk is skipped without a checker call; the middle of a long result is checked as fully as its start and end.
5. Chunks are checked one after another, one judgment per chunk. A failure stops checking; a hit stops it under `withhold`, while `mark` checks every chunk, because the model reads all of the marked text.
6. Clean: the model reads the result unchanged when checking changed nothing; otherwise it reads the checked text, with credentials masked, tags neutralised, and, for a cut result, one line saying how many characters it had and that the rest was removed unchecked.
7. Hit: `withhold` replaces the text with a short notice naming the tool, without any of its text; `mark` wraps the checked text between begin and end lines carrying a random per-call token, so the text cannot fake the end line.
   A hit on an OAuth connection prompt, as JSON or as the repr of a dict result, is withheld even with `mark`, because MindRoom keeps such a control result whole when a replacement still contains it.
8. Each check is logged through the plugin logger with the tool, agent, characters checked, chunk count, verdict (`clean`, `hit`, or `undecided`), and action; the plugin's log line never contains result text.

The hook fails closed.
When the checker fails or times out, has no backend (such as a missing credential), the redactor fails, a chunk cannot be sent verbatim or still cannot fit after splitting, or the settings are invalid, the hook raises, and MindRoom withholds the result: the model reads `The result of <tool> was withheld because a result check failed.`
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
| `judgment` | required | A MindRoom [judgment backend](https://docs.mindroom.chat/configuration/threads/#judgment-backends), such as `{provider: llm, model: <alias from models:>, timeout_seconds: 10}`. Required, so the deployment chooses and pays for the checker model. |
| `tools` | see below | `fnmatch` globs; a call matches when one matches its function name or its toolkit name. |
| `max_chars` | `50000` | Characters of a result that are checked and kept; at most `60000`, the plugin's cap, which with the redactor's lookahead stays under the 65,536 characters MindRoom's redactor reads. |
| `action` | `withhold` | `withhold` replaces a hit with a notice; `mark` wraps it as untrusted data. |

```yaml
models:
  checker:
    provider: openai
    id: your-small-fast-model

plugins:
  - path: plugins/injection-guard-plugin
    required: true  # refuse to start, or reload, with the guard skipped
    settings:
      judgment:
        provider: llm
        model: checker
        timeout_seconds: 10
      action: withhold
      # tools: ["gmail", "website", "mcp_*", "my_fetch_*"]
    hooks:
      injection_guard:
        timeout_ms: 60000  # covers a 50,000-character result at up to 10 s per chunk
```

Default `tools`, the built-in toolkits whose results carry outside text:

- mail and calendar: `gmail`, `outlook_mail`, `google_calendar`, `outlook_calendar`
- web fetch, scrape, and search: `website`, `browser`, `browser_mcp`, `web_browser_tools`, `browserbase`, `crawl4ai`, `firecrawl`, `jina`, `newspaper`, `trafilatura`, `spider`, `scrapegraph`, `agentql`, `brightdata`, `oxylabs`, `serper`, `serpapi`, `searxng`, `tavily`, `exa`, `linkup`, `duckduckgo`, `googlesearch`, `baidusearch`, `wikipedia`, `arxiv`, `pubmed`, `google_scholar`, `hackernews`, `reddit`, `youtube`, `x`
- shared documents: `google_drive`, `google_docs`, `google_sheets`, `notion`, `confluence`, `microsoft_files`, `microsoft_sharepoint`, `microsoft_onenote`
- chat networks the agent reads: `slack`, `discord`, `microsoft_teams`
- every enabled MCP server: `mcp_*`

Opt-in candidates, left out of the defaults:

- `matrix_message`: thread messages reach the model without any tool, so checking only this tool's reads protects nothing consistently.
- `github`: issues and pull requests carry outside text, but agents read code and diffs there constantly; add it where untrusted contributors write.
- `custom_api`: a generic HTTP client whose results are whatever the configured API returns; add it when that API serves outside text.
- `telegram`, `webex`: bot APIs that mostly send; their few reads (`get_chat`, `list_rooms`) return chat metadata.

Setting `tools` replaces the whole list.
Function-name globs are exact.
Toolkit matching is an approximation that errs toward checking more: the result context carries only the function name, so a function matches every configured toolkit (built-in or plugin) that declares that name, such as `send_message` for Discord, Telegram, and Webex.
An MCP function, named `<prefix>_<remote tool>`, belongs to the toolkit `mcp_<server id>` of each enabled server in `mcp_servers` whose `tool_prefix` (by default the server id) it starts with.

Chunks are checked one after another, so a long result takes several checker round trips: a 50,000-character result of prose is about four chunks, and text with many multi-byte characters or escapes needs more.
The hook's timeout covers all of a result's chunks, and the default is 30 seconds; a fast checker answers a chunk in 1 to 2 seconds.
When chunks times the judgment `timeout_seconds` can exceed the hook timeout, raise `timeout_ms` under the plugin entry's `hooks:`, as above; a hook that times out withholds the result.
Checks share MindRoom's judgment capacity (8 requests at once across the process, one per call here), which refuses rather than queues, so a burst of guarded calls can fail checks and withhold results.

## Install

Vendor this plugin with the MindRoom CLI:

```bash
mindroom plugins install injection-guard-plugin
```

Then reference it from `config.yaml` with its settings, as in the example above:

```yaml
plugins:
  - path: plugins/injection-guard-plugin
    required: true
    settings:
      judgment: {provider: llm, model: checker, timeout_seconds: 10}
```

Update to the latest commit later with:

```bash
mindroom plugins update injection-guard-plugin
```

The command pins the exact installed commit in `.mindroom-plugin.lock.json` and strictly validates the plugin before activating it.
The plugin needs plugin API 1.32 (`tool:result_transform`) and is tested against MindRoom v2026.11.436 and later; an older MindRoom skips it with a clear reason.
`mindroom plugins check` warns that it imports judgment, redaction, registry, OAuth, and MCP helpers from outside the stable plugin API; re-run the check after each MindRoom upgrade.

## Data and privacy

- Matched result text, after credential masking, is sent to the configured checker provider, once per chunk.
- With `debug.log_llm_requests: true` (off by default), MindRoom logs the checker's request bodies to disk like any other model request, including text from results that were then withheld.
- Secrets in guarded results reach the model masked, as the checker read them.

## Limits

- Successful result text only: failed tool outcomes, including the text of an MCP server's `isError` result, never reach this hook.
- Withholding replaces the text only; images, audio, video, and files attached to the result still reach the model unchecked.
- Progress a tool streams while it runs is not checked.
- A delegated agent's answer to the agent that delegated to it is not checked; the delegated agent's own tool calls are.
- A result that stops the run after the call, such as an immediate model switch, cannot be told apart by the hook, and MindRoom keeps it whole under `mark`; these come from control tools, not the outside-text readers guarded here.
- A checker can miss an attack or flag benign text, such as an article about prompt injection; `mark` keeps flagged text readable when false hits matter more than a stricter `withhold`.
- The 200-character overlap helps with an instruction cut at a chunk boundary; it does not guarantee that a long instruction is seen whole in one chunk.
- A result longer than `max_chars` is cut, and the model never reads the removed part.
- A checker error or timeout withholds the result.
- Cost and latency, measured with GPT-6 Luna as the checker ($0.10 per million input and $0.50 per million output tokens): a short result is one chunk of about 400 to 500 input and 40 output tokens, about $0.00006 and 1.5 seconds; a 50,000-character result is about $0.0015 and 6 to 8 seconds. Claude Haiku 5.5 cost about $0.0023 for the same 50,000 characters.
