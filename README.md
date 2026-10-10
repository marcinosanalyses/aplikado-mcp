# Aplikado MCP

MCP server for [Aplikado](https://app.aplikado.com). It searches job offers across Polish and EU job boards, ranks them against your CV, and checks how well a CV fits one offer.

- Endpoint: `https://mcp.aplikado.com/mcp` (Streamable HTTP, stateless)
- Auth: OAuth 2.1. You sign in with Google on the Aplikado consent page. 
- Billing: tool calls use the credits of your Aplikado plan, at the same cost as in the app

## Install in Claude Code

Install the server in the folder or repo where you work on your job search, not globally. The easiest way is to ask Claude. Open Claude Code in that folder and paste:

```text
Install the Aplikado MCP server for this folder only (local scope, not user scope):
name "aplikado", HTTP transport, URL https://mcp.aplikado.com/mcp
```

Or run the command yourself in that folder:

```bash
claude mcp add --transport http --scope local aplikado https://mcp.aplikado.com/mcp
```

With `--scope local`, the server is available only in this folder and only to you. To share it with others who work in the repo, use `--scope project` instead. This writes `.mcp.json`, which you can commit.

Then, in Claude Code, run `/mcp`, select `aplikado`, and choose **Authenticate**. Claude Code opens the sign-in page in your browser. If it does not open, copy the URL that Claude Code shows and paste it into your browser. There, sign in with the Google account you use on [app.aplikado.com](https://app.aplikado.com) (Google SSO) and approve access.

If the `aplikado` tools do not show up after you authenticate, reload the window (VS Code: **Developer: Reload Window**) or restart Claude Code.

Claude.ai / Claude Desktop: **Settings → Connectors → Add custom connector**, URL `https://mcp.aplikado.com/mcp`.

Requirements: an Aplikado account. To create one, sign in with Google at [app.aplikado.com](https://app.aplikado.com). Admin and sourcing accounts are refused.

## Tools

### `search_jobs`

Starts a search, waits up to ~45 s, and returns the jobs ranked by fit with the CV.

| Arg | Type | Notes |
|---|---|---|
| `queries` | `[{keyword, secondarySkills?}]`, 1–5 | **Required.** `keyword` ≤120 chars. `secondarySkills` ≤2: they rank matching offers higher and do not filter. Each distinct keyword costs credits. |
| `cv` | `CV` | **Required.** See [CV](#cv). |
| `location` | string | City or country. Leave out for remote jobs. |
| `isRemote` | bool | With `location`: only its remote offers. |
| `languages` | `("en"\|"pl")[]` | Posting languages. Default: both. |
| `salaryMin` | int | Minimum monthly salary. |
| `salaryCurrency` | `PLN\|EUR\|USD\|GBP\|CHF` | |
| `seniority` | `any\|exclude-junior\|only-junior` | Default `any`. |
| `hoursOld` | int 1–720 | Default 168 (7 days). |

Returns: `status` (`running`/`done`), `searchId`, `platformsDone`, `totalFound`, `failedPlatforms`, `jobs[]`, `hiddenLowMatch`, `fitJudgedBy` (`ai`/`text`).

Each item in `jobs[]` has these fields: `matchScore` (0–100; jobs below 20 are left out), `title`, `company`, `location`, `salary`, `workMode`, `skills`, `postedDate`, `platform`, `url`, `alsoListedAt[]`, `matchedKeyword`.

If `status` is `running`, call `get_search_results` again after ~2 min. A search can take several minutes.

### `search_jobs_for_linkedin_profile`

Works like `search_jobs`, but it reads the CV from a public LinkedIn profile.

| Arg | Type | Notes |
|---|---|---|
| `linkedinUrl` | string | **Required.** `https://www.linkedin.com/in/<name>` |
| *(all `search_jobs` filters except `cv`)* | | |

Returns the `search_jobs` result plus `cv`, the data read from the profile. Pass this `cv` to `get_search_results`. Reading the profile costs extra credits and can take up to 2 min. The tool does not keep the name or contact details.

### `get_search_results`

Gets the results of a running or finished search. This tool is read-only. Jobs whose fit is already judged are free.

| Arg | Type | Notes |
|---|---|---|
| `searchId` | uuid | **Required.** From `search_jobs`. |
| `keyword` | string | Same as in the search. |
| `secondarySkills` | string[] ≤2 | Same as in the search. |
| `cv` | `CV` | Same as in the search. Without it, the jobs are not scored. |

### `check_matching_risks`

Checks how well a CV fits one job offer. This is the same check as **Run Match** in the app. It can take 1–2 min.

| Arg | Type | Notes |
|---|---|---|
| `jobUrl` | string ≤500 | **Required.** Link to the job offer. |
| `cv` | `CVMatch` | Give `cv` or `linkedinUrl`. See [CVMatch](#cvmatch). |
| `linkedinUrl` | string | Give it when there is no `cv`. |
| `jobText` | string ≤16000 | The pasted job description. Use only after the error `job_not_found`. |

Returns these fields:
- `job` `{title, company, url}`
- `matchScore` (0–100)
- `riskLevel` (`low`/`medium`/`high`; `null` with a `feedbackNote` if the CV review was not available)
- `summary`
- `risks[]` `{severity, issue, fix}` (≤5)
- `missingSkills[]` (≤10)
- `missingTools[]` (≤10)
- `notes[]` (≤5)

If the job offer is in English and the CV is in Polish, the tool stops before it charges credits and returns `language_mismatch`.

## CV schemas

Do not put a name, email, phone number or links in a CV. The schemas have no fields for them.

### CV

| Field | Type |
|---|---|
| `role` | string ≤120: the current or target job title |
| `pastRoles` | string[] ≤15 |
| `skills` | string[] ≤80 |
| `tools` | string[] ≤80 |
| `languages` | string[] ≤10, with a level, e.g. `"English C1"` |

You must give at least one of `role`, `skills` or `tools`.

### CVMatch

All `CV` fields, plus these fields:

| Field | Type |
|---|---|
| `summary` | string ≤2000 |
| `experiences` | `[{title, company?, dates?, responsibilities?}]` ≤15, most recent first. `responsibilities` has one bullet per line, ≤4000 chars. |
| `education` | string ≤1500 |
| `certifications` | string ≤1500 |

## Prompts (Claude Code `/` menu)

| Command | Args (all optional) |
|---|---|
| `/mcp__aplikado__find_jobs` | `keyword`, `location`, `min_salary`, `posted_within_hours` |
| `/mcp__aplikado__check_matching_risks` | `job_url` |

If an argument is missing, the assistant asks you for it.


## Limits and privacy

- Rate limit: 60 requests/min per user. Over the limit, you get HTTP 429 with `Retry-After: 60`.
- The server does not log tool arguments.
