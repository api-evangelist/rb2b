# @rb2b/rb2b-apis-mcp

MCP (Model Context Protocol) server that exposes [RB2B API's](https://ui.api.rb2b.com) identity resolution and enrichment API as tools for Claude and other MCP-compatible AI assistants.

Resolve IP addresses to companies, look up LinkedIn profiles from emails, find contact information from LinkedIn slugs, and more — all without leaving your AI assistant.

---

## Requirements

- **An MCP-compatible AI client** — [Claude Desktop](https://claude.ai/download) or [Claude Code](https://claude.ai/code) must be installed before setting up this server
- Node.js 18+
- An [RB2B APIs](https://ui.api.rb2b.com) account and API key. Don't have an account? Get your first 100 credits for just $9.

---

## Quick Start

### 1. Install Claude Desktop or Claude Code

This server requires an MCP-compatible client. Install one first:

- **[Claude Desktop](https://claude.ai/download)** — the Claude desktop app for macOS and Windows
- **[Claude Code](https://claude.ai/code)** — the Claude CLI for developers

### 2. Run the setup wizard

```bash
npx -y @rb2b/rb2b-apis-mcp init
```

This will:
- Prompt for your RB2B API key and validate it
- Store it securely in `~/.rb2b/config.json` (permissions: `600`)
- Automatically register the server with Claude Code and Claude Desktop if detected

That's it. No manual config editing needed in most cases.

### Manual registration (if needed)

If auto-registration didn't run or you're using a different MCP client:

**Claude Code:**
```bash
claude mcp add rb2b -- npx -y @rb2b/rb2b-apis-mcp
```

**Claude Desktop** — edit your config file:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "rb2b": {
      "command": "npx",
      "args": ["-y", "@rb2b/rb2b-apis-mcp"]
    }
  }
}
```

Restart Claude Desktop after saving.

---

## Tools

Each API call deducts credits from your account balance. Free tools never consume credits. The server checks your balance before every paid call and will refuse the request if you have insufficient credits.

**Credit warnings** are automatically appended to responses when your balance falls to 500, 200, 100, 50, or 10 credits. Severity levels: `[NOTICE]` (500–51), `[WARNING]` (50–11), `[CRITICAL]` (10 and below).

**Rate limit:** 50 requests/second per endpoint. The server surfaces a clear error if this is exceeded.

### Account

| Tool | Inputs | Cost | Description |
|------|--------|------|-------------|
| `help` | — | free | List all available tools with descriptions and costs |
| `check_credits` | — | free | Check remaining API credits for the account |
| `set_api_key` | `api_key` | free | Update the stored API key (validates before saving) |

### IP Address Lookup

| Tool | Inputs | Cost | Description |
|------|--------|------|-------------|
| `ip_to_company` | `ip_address` | 1 credit | Identify the company associated with an IP address |
| `ip_to_hem` | `ip_address` | 1 credit | Get hashed emails (MD5 + SHA256) linked to an IP address |
| `ip_to_maid` | `ip_address` | 1 credit | Get Mobile Advertising IDs (MAIDs) linked to an IP address |

### Email → Identity

| Tool | Inputs | Cost | Description |
|------|--------|------|-------------|
| `email_to_linkedin_slug` | `email` | 1 credit | Get the LinkedIn slug for a plain-text email |
| `email_to_best_linkedin` | `email` | 1 credit | Get the highest-confidence LinkedIn profile URL for an email |
| `email_to_business_profile` | `email` | 2 credits | Get business/employer profile (title, company, industry) for an email |
| `email_to_maid` | `email` | 1 credit | Get Mobile Advertising IDs linked to an email |

### Hashed Email (HEM) → Identity

Accepts MD5 hashes of email addresses.

| Tool | Inputs | Cost | Description |
|------|--------|------|-------------|
| `hem_to_linkedin_slug` | `md5` | 1 credit | Get the LinkedIn slug for a hashed email |
| `hem_to_best_linkedin` | `md5` | 1 credit | Get the highest-confidence LinkedIn profile URL for a hashed email |
| `hem_to_business_profile` | `md5` | 2 credits | Get business/employer profile for a hashed email |
| `hem_to_maid` | `md5` | 1 credit | Get Mobile Advertising IDs linked to a hashed email |

### LinkedIn → Identity

Accepts LinkedIn vanity URL slugs (e.g. `john-doe-123` from `linkedin.com/in/john-doe-123`).

| Tool | Inputs | Cost | Description |
|------|--------|------|-------------|
| `linkedin_to_hashed_emails` | `linkedin_slug` | 1 credit | Get all hashed emails (personal + business, MD5 + SHA256) for a profile |
| `linkedin_to_best_personal_email` | `linkedin_slug` | 1 credit | Get the highest-confidence personal email for a profile |
| `linkedin_to_personal_email` | `linkedin_slug` | 1 credit | Get all personal email candidates for a profile |
| `linkedin_to_mobile_phone` | `linkedin_slug` | 3 credits | Get the mobile phone number for a profile |
| `linkedin_to_business_profile` | `linkedin_slug` | 4 credits | Get business/employer profile (title, company, industry) for a profile |

---

## Configuration

The API key is stored at `~/.rb2b/config.json` with permissions `600` (owner read/write only):

```json
{
  "apiKey": "your-api-key-here"
}
```

**To update your API key**, use any of these methods:

- **From Claude:** ask Claude to use the `set_api_key` tool — it validates the key before saving and takes effect immediately, no restart needed
- **From the terminal:** run `npx -y @rb2b/rb2b-apis-mcp init` — detects an existing key and prompts for a replacement

---

## Example Usage

Once the server is running, you can ask Claude things like:

- *"Who is visiting my site from IP 203.0.113.42?"*
- *"Find the LinkedIn profile for jane@example.com"*
- *"What company does robbclarke work at?"*
- *"Get the personal email for LinkedIn profile john-doe-123"*
- *"Check my RB2B credit balance"*

Claude will automatically select and call the appropriate tool.


---

## License

MIT
