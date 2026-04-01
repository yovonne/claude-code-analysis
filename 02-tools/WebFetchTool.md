# WebFetchTool

## Purpose

WebFetchTool fetches and extracts content from specified URLs, converting HTML to markdown and processing the content through a fast secondary AI model (Haiku) to answer user-specified prompts. It includes a 15-minute LRU cache, domain blocklist preflight checks, redirect safety validation, and binary content persistence. The tool is designed for read-only web content retrieval with multiple security layers to prevent data exfiltration and access to internal resources.

## Location

- `restored-src/src/tools/WebFetchTool/WebFetchTool.ts` — Main tool definition and execution logic (319 lines)
- `restored-src/src/tools/WebFetchTool/utils.ts` — URL fetching, caching, domain checks, HTML-to-markdown conversion (531 lines)
- `restored-src/src/tools/WebFetchTool/prompt.ts` — Tool description and secondary model prompt generation (47 lines)
- `restored-src/src/tools/WebFetchTool/preapproved.ts` — Preapproved host/domain list (167 lines)
- `restored-src/src/tools/WebFetchTool/UI.tsx` — Terminal UI rendering (72 lines)

## Key Exports

### From WebFetchTool.ts

| Export | Description |
|--------|-------------|
| `WebFetchTool` | The complete tool definition built via `buildTool()` |
| `Output` | Zod-inferred output type: `{ bytes, code, codeText, result, durationMs, url }` |

### From utils.ts

| Export | Description |
|--------|-------------|
| `getURLMarkdownContent(url, abortController)` | Fetches URL content, converts HTML to markdown, returns cached or fresh content |
| `applyPromptToMarkdown(prompt, content, signal, ...)` | Sends content + prompt to Haiku model for processing |
| `validateURL(url)` | Validates URL format, length, and safety (no credentials, public hostname) |
| `checkDomainBlocklist(domain)` | Checks domain against Anthropic's blocklist API |
| `isPermittedRedirect(originalUrl, redirectUrl)` | Validates redirect safety (same origin, www. variance only) |
| `getWithPermittedRedirects(url, signal, redirectChecker)` | HTTP GET with controlled redirect following |
| `isPreapprovedUrl(url)` | Checks if URL is on the preapproved domain list |
| `clearWebFetchCache()` | Clears both URL and domain check caches |
| `FetchedContent` | Type: `{ content, bytes, code, codeText, contentType, persistedPath?, persistedSize? }` |

### From preapproved.ts

| Export | Description |
|--------|-------------|
| `PREAPPROVED_HOSTS` | Set of ~100+ preapproved domains for code-related content |
| `isPreapprovedHost(hostname, pathname)` | Checks if host/path is preapproved (supports path-scoped entries) |

## Input/Output Schemas

### Input Schema

```typescript
{
  url: string,     // Required: The URL to fetch content from
  prompt: string,  // Required: The prompt to run on the fetched content
}
```

### Output Schema

```typescript
{
  bytes: number,       // Size of the fetched content in bytes
  code: number,        // HTTP response code
  codeText: string,    // HTTP response code text
  result: string,      // Processed result from applying the prompt to the content
  durationMs: number,  // Time taken to fetch and process the content
  url: string,         // The URL that was fetched
}
```

## Content Fetching Pipeline

### Execution Flow

```
1. INPUT RECEIVED
   - Model returns WebFetch tool_use with { url, prompt }

2. VALIDATION
   - validateInput(): checks URL is parseable
   - URL validated: length ≤ 2000, no username/password, public hostname (≥ 2 parts)

3. PERMISSION CHECK
   - checkPermissions():
     a. Preapproved host check → auto-allow
     b. Deny rule lookup → deny
     c. Ask rule lookup → ask (with suggestions)
     d. Allow rule lookup → auto-allow
     e. No rule → ask (default)

4. CONTENT FETCHING (getURLMarkdownContent)
   a. Cache lookup (LRU, 15-min TTL, 50MB max)
   b. URL validation (validateURL)
   c. HTTP→HTTPS upgrade if needed
   d. Domain blocklist preflight check (api.anthropic.com)
   e. HTTP GET with controlled redirects (max 10 hops)
   f. Content type detection (HTML vs text vs binary)
   g. HTML→Markdown conversion via Turndown
   h. Binary content persistence to disk
   i. Cache storage

5. PROMPT APPLICATION
   a. Preapproved + markdown + <100K chars → return raw content
   b. Otherwise → send to Haiku model with prompt
   c. Content truncated to 100K chars before model call

6. RESULT PROCESSING
   - Output assembled with bytes, code, result, duration
   - Binary content path appended if persisted
   - Result returned to model
```

### Caching Strategy

Two-tier LRU caching:

| Cache | Key | TTL | Max Size | Purpose |
|-------|-----|-----|----------|---------|
| `URL_CACHE` | Full URL | 15 minutes | 50 MB | Fetched content |
| `DOMAIN_CHECK_CACHE` | Hostname | 5 minutes | 128 entries | Blocklist check results |

The domain check cache stores only `allowed` status — blocked/failed entries are re-checked on each attempt. This prevents redundant HTTP round-trips when fetching multiple paths from the same domain.

### Redirect Handling

Redirects are handled with strict safety controls:

- **Max hops**: 10 (prevents infinite loops)
- **Permitted redirects**: Only same-origin redirects (www. variance allowed)
- **Protocol must match**: No http→https or https→http redirects
- **Port must match**: No port changes
- **No credentials**: Redirect URLs with usernames/passwords are rejected
- **Cross-host redirects**: Returned to model as a `REDIRECT DETECTED` message with instructions to re-fetch

## Security Controls

### URL Validation

| Check | Rule |
|-------|------|
| Length | ≤ 2000 characters |
| Format | Must be a valid, parseable URL |
| Credentials | No username or password in URL |
| Hostname | Must be publicly resolvable (≥ 2 dot-separated parts) |
| Protocol | HTTP automatically upgraded to HTTPS |

### Domain Blocklist Preflight

Before fetching, the hostname is checked against Anthropic's domain blocklist API:

```
GET https://api.anthropic.com/api/web/domain_info?domain=<hostname>
```

- **allowed** (`can_fetch: true`): Proceed with fetch, cache result
- **blocked** (`can_fetch: false`): Throw `DomainBlockedError`
- **check_failed**: Throw `DomainCheckFailedError`
- **Skipped**: If `settings.skipWebFetchPreflight` is true (enterprise customers with restrictive security policies)

### Preapproved Hosts

~100+ domains are preapproved for automatic access without permission prompts. Categories include:

- **Anthropic**: platform.claude.com, modelcontextprotocol.io, etc.
- **Programming languages**: docs.python.org, developer.mozilla.org, doc.rust-lang.org, etc.
- **Frameworks**: react.dev, angular.io, django, flask, spring, etc.
- **Cloud/DevOps**: docs.aws.amazon.com, kubernetes.io, docker.com, etc.
- **Databases**: postgresql.org, mongodb.com, redis.io, etc.
- **Data/ML**: huggingface.co, kaggle.com, pytorch.org, etc.

Path-scoped entries (e.g., `github.com/anthropics`) enforce path segment boundaries to prevent prefix-matching attacks.

### Egress Proxy Detection

Detects enterprise egress proxy blocks via the `X-Proxy-Error: blocked-by-allowlist` header and throws `EgressBlockedError` with structured error information.

### Content Length Limits

| Limit | Value | Purpose |
|-------|-------|---------|
| `MAX_URL_LENGTH` | 2000 chars | Prevent oversized URLs |
| `MAX_HTTP_CONTENT_LENGTH` | 10 MB | Prevent resource exhaustion |
| `MAX_MARKDOWN_LENGTH` | 100K chars | Truncate content before Haiku model call |
| `FETCH_TIMEOUT_MS` | 60 seconds | Prevent hanging on slow servers |
| `DOMAIN_CHECK_TIMEOUT_MS` | 10 seconds | Prevent slow blocklist checks |
| `MAX_REDIRECTS` | 10 | Prevent redirect loops |

### Binary Content Handling

Binary content (PDFs, images, etc.) is:
1. Saved to disk with a MIME-derived extension
2. Still decoded to UTF-8 and sent to Haiku (PDFs have enough ASCII structure for summarization)
3. The persisted file path is appended to the result so the model can inspect the raw file

## Permission Handling

### Permission Decision Logic

```
1. Is host preapproved? → allow (reason: "Preapproved host")
2. Is there a deny rule for this domain? → deny
3. Is there an ask rule for this domain? → ask (with suggestion to allow)
4. Is there an allow rule for this domain? → allow
5. No rule matched → ask (with suggestion to allow)
```

Permission rules are keyed by `domain:<hostname>`, derived from the input URL.

### Suggestion System

When permission is requested, the system suggests adding an allow rule:
```json
{
  "type": "addRules",
  "destination": "localSettings",
  "rules": [{ "toolName": "WebFetch", "ruleContent": "domain:<hostname>" }],
  "behavior": "allow"
}
```

## Prompt Application

### Secondary Model Processing

For non-preapproved content or content exceeding size limits, the fetched markdown is processed through the Haiku model:

```
Web page content:
---
<markdown content (truncated to 100K chars)>
---

<user prompt>

<guidelines based on preapproved status>
```

**Preapproved domain guidelines**: Concise response based on content, including code examples and documentation excerpts.

**Non-preapproved domain guidelines**: Strict 125-character max for quotes, quotation marks for exact language, no commentary on legality, no song lyrics.

## Configuration

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `USER_TYPE` | `'ant'` logs hostname analytics for internal monitoring |

### Settings

| Setting | Type | Default | Description |
|---------|------|---------|-------------|
| `skipWebFetchPreflight` | boolean | false | Skip the domain blocklist check (enterprise use) |

## Dependencies

### Internal

| Module | Purpose |
|--------|---------|
| `services/api/claude.js` | Haiku model queries |
| `utils/mcpOutputStorage.js` | Binary content persistence |
| `utils/settings/settings.js` | Settings access |
| `utils/http.js` | User-Agent generation |
| `utils/format.js` | File size formatting |
| `utils/permissions/permissions.js` | Permission rule lookup |
| `services/analytics/` | Event logging |

### External

| Package | Purpose |
|---------|---------|
| `axios` | HTTP client for fetching URLs |
| `lru-cache` | LRU cache implementation |
| `turndown` | HTML to Markdown conversion |
| `zod/v4` | Input/output schema validation |
| `react` | UI rendering |

## Data Flow

```
Model Request
    |
    v
WebFetchTool Input { url, prompt }
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ VALIDATION                                                      │
│ - validateInput(): URL parseable                                │
│ - validateURL(): length ≤ 2000, no credentials, public hostname │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PERMISSION CHECK                                                │
│  1. Preapproved host → allow                                    │
│  2. Deny rule → deny                                            │
│  3. Ask rule → ask                                              │
│  4. Allow rule → allow                                          │
│  5. No rule → ask                                               │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ CONTENT FETCH (getURLMarkdownContent)                           │
│                                                                 │
│  1. Cache lookup (URL_CACHE, 15-min TTL, 50MB max)              │
│  2. HTTP→HTTPS upgrade                                          │
│  3. Domain blocklist check (api.anthropic.com)                  │
│  4. HTTP GET with redirect control (max 10 hops)                │
│  5. Content type detection                                      │
│  6. HTML→Markdown (Turndown) or raw passthrough                 │
│  7. Binary persistence to disk                                  │
│  8. Cache storage                                               │
└─────────────────────────────────────────────────────────────────┘
    |
    v
┌─────────────────────────────────────────────────────────────────┐
│ PROMPT APPLICATION                                              │
│                                                                 │
│  Preapproved + markdown + <100K → return raw                    │
│  Otherwise → Haiku model with prompt                            │
│  Content truncated to 100K chars                                │
└─────────────────────────────────────────────────────────────────┘
    |
    v
Output { bytes, code, codeText, result, durationMs, url }
    |
    v
Model Response (tool_result block)
```
