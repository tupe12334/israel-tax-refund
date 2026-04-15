# Israel Tax Refund — AI-Assisted Filing

An agentic Claude Code workspace that automates the Israeli tax refund process. Clone the repo, open a Claude session, and let Claude navigate the tax authority website on your behalf using Playwright.

## How It Works

1. You clone this repo and open it in Claude Code
2. Claude guides you through entering your personal and financial details
3. A Playwright MCP server controls a real browser session
4. Claude navigates the Israeli Tax Authority ([Misim](https://www.misim.gov.il)) portal, fills in your information, and submits the refund request on your behalf

No coding required on your end.

## Prerequisites

| Requirement                                                   | Details                                                                       |
| ------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| [Claude Code](https://claude.ai/code)                         | Desktop app, VS Code extension, or CLI (`npm i -g @anthropic-ai/claude-code`) |
| [Playwright MCP](https://github.com/microsoft/playwright-mcp) | Browser automation MCP server                                                 |
| Node.js 18+                                                   | Required by Playwright MCP                                                    |
| Israeli ID (ת.ז.)                                             | Needed to authenticate on the tax portal                                      |

## Setup

### 1. Clone the repo

```bash
git clone https://github.com/tupe12334/israel-tax-refund.git
cd israel-tax-refund
```

### 2. Install Playwright MCP

```bash
npm install -g @playwright/mcp
```

### 3. Configure the Playwright MCP server in Claude Code

Add the following to your Claude Code MCP settings (`~/.claude/settings.json`):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp"]
    }
  }
}
```

Or run the setup command inside Claude Code:

```
/mcp add playwright npx @playwright/mcp
```

### 4. Open Claude Code in this directory

```bash
claude .
```

## Usage

Once inside Claude Code, simply tell Claude:

> "Help me apply for a tax refund"

Claude will:

- Ask for the required personal information (ID number, income details, bank account for refund)
- Open a browser via Playwright
- Log in to the tax authority portal using your credentials
- Fill out and submit the refund form
- Report back with confirmation details

## What You'll Need to Have Ready

- Israeli ID number (מספר זהות)
- Tax year you are claiming for
- Income details (תלושי שכר / Form 106)
- Bank account number for the refund deposit
- Access to your email/phone for OTP verification during login

## Privacy & Security

- Your personal data is only used during the active Claude session and is never stored in this repository
- The browser session runs locally on your machine via Playwright
- No credentials are sent to any third-party service — Claude interacts with the official government portal directly through your local browser

## Troubleshooting

**Playwright MCP not connecting**
Make sure the MCP server is listed under `/mcp` in Claude Code and shows a green status. If not, restart Claude Code after updating `settings.json`.

**Login OTP not arriving**
The tax authority portal sends OTP to the phone number registered on your ID. Make sure your details are up to date at [misim.gov.il](https://www.misim.gov.il).

**Session timeout**
Government portals often have short session windows. If Claude loses the session mid-flow, ask it to restart the browser and try again.

## Contributing

Improvements to the CLAUDE.md instructions, skills, or Playwright flow are welcome. Open a PR or issue on [tupe12334/israel-tax-refund](https://github.com/tupe12334/israel-tax-refund).

## Disclaimer

This tool automates interaction with publicly accessible government web portals on your behalf. You are responsible for the accuracy of the information you provide. This is not legal or tax advice.
