# Indeed MCP Server

<!-- mcp-name: com.hasdata/indeed -->

A hosted Model Context Protocol (MCP) server that gives Claude, Cursor, Windsurf and any other MCP client two read-only Indeed tools. Search job listings by keyword and location, and read a single posting in full, all as structured JSON, with no Indeed developer account and no partner approval.

It reads public job postings that a signed-out visitor can see.

```
https://mcp.hasdata.com/api/mcp?apis=indeed
```

[![Glama score](https://glama.ai/mcp/servers/HasData/indeed-mcp/badges/score.svg)](https://glama.ai/mcp/servers/HasData/indeed-mcp)
[![tool contract](https://github.com/HasData/indeed-mcp/actions/workflows/contract.yml/badge.svg)](https://github.com/HasData/indeed-mcp/actions/workflows/contract.yml)
[![MCP](https://img.shields.io/badge/MCP-remote%20%7C%20streamable%20HTTP-6366f1?style=flat-square)](https://modelcontextprotocol.io)
[![Tools](https://img.shields.io/badge/tools-2-10b981?style=flat-square)](#tools)
[![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)](LICENSE)

## Contents

- [What you need](#what-you-need)
- [Quick start](#quick-start)
- [Example prompts](#example-prompts)
- [Tools](#tools)
- [Errors and failure paths](#errors-and-failure-paths)
- [Pricing, free tier and limits](#pricing-free-tier-and-limits)
- [Tool selection](#tool-selection)
- [How it compares](#how-it-compares)
- [FAQ](#faq)
- [HasData links](#hasdata-links)
- [Development](#development)
- [Contributing](#contributing)
- [License](#license)

## What you need

An MCP client and a HasData API key from the [dashboard](https://app.hasdata.com/sign-up?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp), free to create with no card, and the trial covers about 200 calls at the 5-credit rate. This is a remote server, so the simplest path is a URL and an `x-api-key` header, with no container to run and no Indeed developer account anywhere in the flow. A client that only speaks stdio reaches it through a thin launcher, published as `@hasdata/indeed-mcp` on npm and `hasdata-indeed-mcp` on PyPI, shown below.

## Quick start

The server URL is the same for every client. We run it hands-on in Claude Code and Claude Desktop. The other blocks follow each client's own documented format for a remote server.

| Field | Value |
| :--- | :--- |
| URL | `https://mcp.hasdata.com/api/mcp?apis=indeed` |
| Transport | HTTP, streamable |
| Auth header | `x-api-key: HASDATA_API_KEY` |

Clients with OAuth support can add the same URL as a connector and sign in without putting a key in a config file.

<details>
<summary><b>Claude Code</b></summary>

```bash
claude mcp add --transport http indeed "https://mcp.hasdata.com/api/mcp?apis=indeed" \
  --header "x-api-key: HASDATA_API_KEY"
```

</details>

<details>
<summary><b>Claude Desktop</b></summary>

Settings, then Connectors, then Add custom connector, then paste `https://mcp.hasdata.com/api/mcp?apis=indeed` and sign in.

For the config-file route, Claude Desktop loads only local (stdio) servers, so it reaches a remote server through a stdio launcher. The `@hasdata/indeed-mcp` package is that launcher, and it reads the key from the environment. Add this to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "indeed": {
      "command": "npx",
      "args": ["-y", "@hasdata/indeed-mcp"],
      "env": { "HASDATA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

For Python instead of Node, swap the launcher for the PyPI package, which `uvx` runs without a manual install:

```json
{
  "mcpServers": {
    "indeed": {
      "command": "uvx",
      "args": ["hasdata-indeed-mcp"],
      "env": { "HASDATA_API_KEY": "YOUR_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>Cursor</b></summary>

`~/.cursor/mcp.json` for every project, or `.cursor/mcp.json` for one:

```json
{
  "mcpServers": {
    "indeed": {
      "url": "https://mcp.hasdata.com/api/mcp?apis=indeed",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>Windsurf</b></summary>

`~/.codeium/windsurf/mcp_config.json`. Windsurf calls the field `serverUrl`, not `url`:

```json
{
  "mcpServers": {
    "indeed": {
      "serverUrl": "https://mcp.hasdata.com/api/mcp?apis=indeed",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

<details>
<summary><b>VS Code</b></summary>

`.vscode/mcp.json` in the workspace:

```json
{
  "servers": {
    "indeed": {
      "type": "http",
      "url": "https://mcp.hasdata.com/api/mcp?apis=indeed",
      "headers": { "x-api-key": "HASDATA_API_KEY" }
    }
  }
}
```

</details>

## Example prompts

Prompts, not code. Paste one in and the agent picks the tool itself. Each is annotated with the calls it takes, because every successful call costs 5 credits.

> Search Indeed for "python developer" jobs in New York sorted by date, and give me the ten most recent with company and salary.

*One call, 5 credits. A listing page carries company, salary and posted date already.*

> Take the top posting from that search and pull its full description and requirements.

*One call, 5 credits. The listing carries a job URL, which the details tool takes directly.*

> Find "data analyst" jobs in Austin, then pull full details on the three that list a salary.

*Four calls, 20 credits. One listing, then one details call for each of the three.*

> Compare the salaries posted for "registered nurse" in Chicago against Houston.

*Two calls, 10 credits, one listing per city.*

Salary is on a listing only when the posting states one, so a "jobs with salary" prompt filters on the field rather than assuming it. Paging costs a call each time, through the `start` offset.

## Tools

Two tools, read-only. Samples below are trimmed from real calls, and the numbers move as Indeed updates. Read them as shapes. Each tool name links to its endpoint reference, which carries the full field list.

The samples are the payload, not the whole response. A `tools/call` result carries one text block, and that text is itself JSON holding `url`, `status`, `text` and `json`, with the scraped data under `json`. From a raw JSON-RPC response the path is `result.content[0].text`, parsed, then `.json`. A chat client unwraps that for you and code talking to the endpoint directly does not.

### Get Indeed job listings

[`hasdata_indeed_listing_getJobListings`](https://docs.hasdata.com/apis/indeed/listing?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp)

A page of search results by keyword and location.

| Parameter | Type | Required | Notes |
| :--- | :--- | :--- | :--- |
| `keyword` | string | yes | The search term, as a candidate would type it |
| `location` | string | yes | City and state, or any location string Indeed accepts |
| `sort` | string | | `relevance` by default, or `date` for newest first |
| `domain` | string | | A country site such as `au.indeed.com`. Defaults to the US site |
| `start` | number | | Result offset for paging, in steps of the page size |

Returns `searchInformation`, a `jobs` array, `peopleAlsoSearchFor`, and `pagination` whose `nextPage` is the URL of the following page. Each job carries `title`, `company`, `location`, `url`, a short `description`, `sponsored`, the relative `date` and the absolute `isoDate`, a `details` array of labels like `Full-time` and `Hybrid work`, a `benefits` array, and a `salary` object when the posting states one.

> A listing carries `details` as an array of plain strings. The job-details tool below returns `details` as an object with `jobType` and `workSetting` arrays. Same name, different shape, so read each per its own tool.

```json
{
  "title": "Hedge Fund Application Developer",
  "company": "TBA",
  "location": "Stamford, CT 06901",
  "url": "https://www.indeed.com/pagead/clk?...",
  "sponsored": true,
  "date": "30+ days ago",
  "isoDate": "2025-06-03T17:22:54.869Z",
  "details": ["Full-time", "Hybrid work"],
  "benefits": ["Health insurance", "401(k)", "401(k) matching"],
  "salary": { "min": 80000, "max": 150000, "type": "YEARLY" }
}
```

### Get Indeed job details

[`hasdata_indeed_job_getJobDetails`](https://docs.hasdata.com/apis/indeed/job?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp)

One posting in full, by its job URL.

| Parameter | Type | Required | Notes |
| :--- | :--- | :--- | :--- |
| `url` | string | yes | A posting URL, the `url` field from a listing result |

Returns `title`, `company`, `location`, `sponsored`, a `details` object with `jobType` and `workSetting` arrays, a `salary` object when present, and the posting body in both `description` (plain text) and `descriptionHtml` (the same content with its list and paragraph markup kept). Use `descriptionHtml` when you want the structure and `description` when you want to feed clean text to a model.

```json
{
  "title": "Python Developer",
  "company": "Think IT Technologies",
  "location": "New York, NY 10114",
  "sponsored": false,
  "details": { "jobType": ["Contract"], "workSetting": ["In-person"] },
  "salary": { "min": 60.5, "max": 65, "type": "HOURLY" },
  "description": "Overview\nWe are seeking a Python Developer...",
  "descriptionHtml": "<p><b>Overview</b></p><p>We are seeking a Python Developer...</p>"
}
```

## Errors and failure paths

Your client almost never sees an HTTP error code from a tool call. The MCP layer answers 200 and puts the failure inside the result, with `isError` set to `true` and the reason as text. The agent reads a message where you might expect a status line.

**A wrong key surfaces as tool output, not as a failed connection.** `tools/list` accepts any non-empty key and returns both tools, so the client completes its handshake and shows green. The first tool call then comes back with `isError: true` and the text `HasData API error: 401 Unauthorized`. Watch for that string, because nothing earlier in the flow reports the problem.

**A missing key is the one real HTTP error.** Authorization runs before any tool, and the connection itself fails with 401. CORS headers are present, and a browser client reads the status and not an opaque network failure.

**An argument that breaks a tool's schema is rejected before it becomes a scrape.** The server answers with `isError: true` and the text `MCP error -32602: Input validation error`, naming the offending field. Nothing is fetched and nothing is charged.

**A search that matches nothing returns a successful result with an empty `jobs` array**, not an error. A keyword and location combination with no openings still comes back with `requestMetadata.status` set to `ok`. Test for the array length before you iterate.

**A posting that has been taken down returns 400** with `requestMetadata.status` set to `error`. Indeed expires listings quickly, so a URL from an old search can be gone.

Results that carry data also carry a `requestMetadata.id` worth quoting in support.

## Pricing, free tier and limits

Each Indeed tool costs **5 credits per successful call**. Response size does not change the price. A listing page with fifty jobs costs the same as one with two.

The free trial is **1,000 credits over 30 days with no card**, which is 200 Indeed calls. After that an active account keeps getting 100 credits topped up each day whenever its balance drops below 100, so a low-volume agent runs on the free tier indefinitely.

Paid plans start at **$49 a month** for 200,000 credits, which is 40,000 calls. The unit price falls with volume, from **$1.23 per 1,000 calls** on the entry plan to **$0.50** on Business, **$0.42** on Growth and **$0.37** on the largest [high-volume plans](https://hasdata.com/prices?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp).

Your plan also sets concurrency. The free trial allows 1 request at a time, Startup 15, Business 30, Growth 50, and the high-volume plans run from 200 to 1,500. Handle the overflow case defensively in anything unattended.

A request that comes back non-200 is not billed. A successful call that finds nothing is still a call.

## Tool selection

The `apis` query parameter decides which tools your agent sees. Fewer tools means less context spent on tool definitions, and fewer chances for the model to reach for the wrong one.

```
?apis=indeed                     the two tools in this repo
?apis=indeed,glassdoor           add Glassdoor jobs
?apis=indeed,google_serp         add Google search
```

The parameter takes provider names like `indeed` and individual API names like `indeed_listing`. Misspelled names are ignored. If every name is wrong the request fails with 400, and the body lists both what it did not recognise and every valid value. Drop the parameter and the same endpoint exposes all 57 HasData tools.

## How it compares

Indeed retired its public Publisher and Job Search APIs, and programmatic access now runs through approved partnerships and ATS integrations rather than a self-serve key. For reading public listings and postings across arbitrary searches, there is no open official route.

| | Official Indeed access | This server |
| :--- | :--- | :--- |
| Access | Partner or ATS approval | One key and one URL |
| Scope | Whatever the partnership grants | Any public search or posting |
| Setup | Business review | None |
| Output | Depends on the integration | Structured JSON, salary pre-parsed |
| Writes | Application flows for approved partners | Read-only, public data only |

**What this server does not do.** No application submission, no employer dashboard, no candidate data, no private postings. It reads what a signed-out visitor can see.

## FAQ

### Is there an official Indeed MCP server?

Indeed does not publish one. This one is maintained by HasData and reads public pages, which is why it needs no Indeed developer account.

### What is an Indeed MCP server?

A server that exposes Indeed job data as tools an AI client can call. The client sends a tool call over the Model Context Protocol, the server fetches the data and returns structured JSON, and the model works with the result. This one exposes two tools and runs remotely.

### Do I need an Indeed API key or partner account?

No. The only credential is your HasData key. There is no Publisher account to apply for, because the tools read public Indeed pages.

### How do I page through more than one screen of results?

Pass the `start` offset to the listing tool. The response also returns `pagination.nextPage` as the URL of the following page, so an agent can walk results without computing offsets by hand.

### Why is a salary sometimes missing?

Because the posting does not state one. The `salary` object is present only when Indeed shows a figure. Read it defensively.

### Can I use this together with other HasData APIs?

Yes. The `apis` parameter takes a list, and `?apis=indeed,glassdoor` gives your agent Indeed plus Glassdoor. [Drop the parameter](#tool-selection) and you get everything.

### Compliance and personal data

HasData accesses publicly available data only. A platform's terms may restrict automated access, and you are responsible for your own compliance. Where the data you collect includes personal information, make sure you have a lawful basis for it under GDPR, CCPA or the equivalent rules in your jurisdiction.

## HasData links

| | |
| :--- | :--- |
| Product page and request builder | [Indeed Scraper API](https://hasdata.com/apis/indeed-api?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp) |
| Server documentation | [MCP server docs](https://docs.hasdata.com/mcp-server?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp) |
| All 57 tools in one server | [HasData/hasdata-mcp](https://github.com/HasData/hasdata-mcp) |
| Client walkthroughs | [MCP clients and integrations](https://hasdata.com/integrations/mcp?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp) |
| Everything else we scrape | [Indeed Scraper API and 54 more](https://hasdata.com/apis/?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp) |
| Plans and credit costs | [Plans and credit costs](https://hasdata.com/prices?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp) |
| Keys and usage | [HasData dashboard](https://app.hasdata.com?utm_source=github&utm_medium=syndication&utm_campaign=indeed-mcp) |

## Development

This repository is configuration and documentation for a remote server. There is no build step and nothing to containerize.

The tests in `test/` assert the tool contract, the part that can break without a commit here. They check that `?apis=indeed` returns exactly two tools, that every tool still declares its required parameters, that no name changed, and that the key in use is actually accepted. That last check calls a tool for real and costs 5 credits, which is the price of a canary that can fail for the right reason.

```bash
# macOS and Linux
HASDATA_API_KEY=your_key_here npm test

# Windows PowerShell
$env:HASDATA_API_KEY="your_key_here"; npm test
```

The same suite runs in CI on every push and once a week on a schedule, because the upstream tool list can change without anyone touching this repository. A failure means the tool list moved, the key stopped working, or the endpoint was unreachable, and the assertion message says which.

## Contributing

Corrections to the tool tables and the response samples are the most useful contribution, because those are the parts that drift. Include the call you made and the response you got. Pull requests from forks run the suite without a key, and the live checks skip instead of going red.

## License

MIT. See [LICENSE](LICENSE).
