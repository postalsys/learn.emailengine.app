# EmailEngine Documentation - Claude Code Guide

This repository contains the **unified Docusaurus documentation site** for EmailEngine, an Email API for IMAP and SMTP.

## Key URLs

- **EmailEngine Homepage:** https://emailengine.app/ (landing page only)
- **GitHub Repository:** https://github.com/postalsys/emailengine (source-available, not open source)
- **Documentation Site:** https://learn.emailengine.app/ (hosts this Docusaurus project)
- **Blog:** https://blog.emailengine.app/ (low activity)

## Project Status

**✅ Production Ready** - All documentation has been unified and cleaned up. The site is ready for deployment.

- **137 authored documentation files** covering all EmailEngine features
- **81 auto-generated API docs** from OpenAPI spec, plus an overview page
- **~67,000 lines** of authored documentation
- **Build status:** ✅ Passing (with minor non-critical anchor warnings)

## Editorial Model: MDN

**Treat this site as MDN Web Docs for EmailEngine.** MDN is the reference model for how these pages are written, maintained, and argued about. When a judgment call comes up that the rules below do not settle, decide it the way an MDN editor would.

What that means in practice:

- **Document what ships, not what is intended.** Every claim traces to the source, the OpenAPI spec, or an observed response. If you cannot verify it, do not write it. A guess that reads well is worse than a gap.
- **Reference before narrative.** The primary job of a page is to let a reader look something up and get an exact answer: the parameter, its type, its default, what it does. Tutorials and guidance are welcome, but they never displace the reference material.
- **Neutral, plain, present tense.** No marketing voice, no "simply", "easily", "just", "powerful", or "seamless". State behavior; let the reader judge it.
- **Behavior notes carry versions.** When something changed, say which version changed it, the way MDN's compatibility notes do. Deprecated and removed features are labeled and kept until they are gone from supported releases, not quietly deleted.
- **One canonical page per concept.** Link to it rather than restating it. Duplicated explanations drift apart, and the divergence is always discovered by a reader.
- **Do not document bugs as features.** If the implementation is wrong, report it rather than writing the defect into the reference. Document the behavior a reader can rely on today.
- **Fix what you touch.** An error noticed on a page being edited for another reason gets fixed in that edit, with the page read whole rather than patched at the point of the change.
- **Examples are complete and correct.** Runnable as written, minimal, and free of placeholders that would fail if pasted.
- **Write for someone who arrived from a search engine.** Assume no knowledge of the preceding page, and no reading of the section above.

MDN's own guidance is the tiebreaker for anything not covered here: https://developer.mozilla.org/en-US/docs/MDN/Writing_guidelines

## Important Rules

**CRITICAL - Read These First:**

1. **Git Commit Messages** - DO NOT include Claude Code attribution or co-authorship

   - ❌ Never add: "Generated with Claude Code"
   - ❌ Never add: "Co-Authored-By: Claude"
   - ✅ Write clear, professional commit messages without AI attribution

2. **No Emojis in Documentation** - Never use emojis in documentation files

   - ❌ Don't use: ✅ ❌ 🚀 💡 ⚠️ etc. in markdown files
   - ✅ Exception: Only if explicitly needed for visual indicators (e.g., alert symbols in admonitions)
   - ✅ Use text instead: "Success", "Warning", "Important", etc.
   - Note: Existing emojis in CLAUDE.md itself are acceptable for this meta-documentation file

3. **Dash Usage in Documentation** - Use a single hyphen-minus as a thought separator

   - ❌ Don't use: `--` (double dash/hyphen), `—` (em dash), `–` (en dash)
   - ✅ Use: `-` (single hyphen-minus) for parenthetical asides and separators
   - Example: "Queue Management - Detailed guide on BullMQ internals"

4. **URL Redirects** - When renaming or moving documentation pages

   - If a documentation page URL changes (file renamed or moved), add a redirect from the old URL to the new one
   - This prevents 404 errors when users access old/bookmarked links
   - Add redirects in `docusaurus.config.ts` under the `@docusaurus/plugin-client-redirects` plugin
   - Example:
     ```typescript
     {
       from: '/docs/old-path/old-page',
       to: '/docs/new-path/new-page',
     },
     ```
   - Only skip redirects if a new page with different content replaces the old URL

5. **AI Agent Reference Files** - Keep these files up to date when API changes

   Two files exist specifically for AI coding assistants and must be updated when the API changes:

   - **`docs/reference/llm-context.md`** - Human-readable AI agent reference document
     - Contains: Complete API endpoint list, webhook events, common patterns, decision trees
     - Update when: New endpoints added, webhooks changed, major features added
     - Location: Appears in the sidebar under "AI Agent Reference"

   - **`static/capabilities.json`** - Machine-readable API capabilities manifest
     - Contains: Structured JSON with all endpoints, webhook events, account types, error codes
     - Update when: Any API surface changes (new endpoints, parameters, webhooks)
     - Accessible at: `/capabilities.json` on the documentation site

   **When to Update These Files:**
   - New API endpoint added
   - Existing endpoint parameters changed
   - New webhook event type added
   - New account type supported
   - New error codes introduced
   - Major feature additions

   **How to Update:**
   1. Check `sources/swagger.json` for the latest API version
   2. Update `docs/reference/llm-context.md` with new endpoints/features in the appropriate tables
   3. Update `static/capabilities.json` with corresponding structured data
   4. Verify the version number in both files matches the current API version
   5. Test that `capabilities.json` is valid JSON: `node -e "require('./static/capabilities.json')"`

   **Why These Files Matter:**
   AI agents (like Claude Code, GitHub Copilot, Cursor) helping developers integrate with EmailEngine use these files to quickly understand the API surface without parsing the full OpenAPI spec or reading dozens of documentation pages.

6. **Screenshot Automation** - When generating screenshots for documentation

   **Environment Variables for Automation:**

   - Set `EENGINE_DISABLE_SETUP_WARNINGS=true` to disable admin password warnings
   - Set `EENGINE_REQUIRE_API_AUTH=false` to allow API calls without access tokens
   - This enables fully automated EmailEngine startup without web sessions or API authentication
   - Without admin password, the dashboard is accessible without logging in
   - Makes it easy to capture screenshots programmatically or via automation tools

   **Starting EmailEngine for Screenshots and Testing:**

   Always run EmailEngine from source for screenshots, testing, and development:

   ```bash
   # Step 0: Clean up any existing processes (prevent port conflicts)
   # Kill any existing EmailEngine processes on port 7003
   lsof -ti:7003 | xargs kill -9 2>/dev/null || true

   # Kill any existing SSH tunnels to kreata.ee
   pkill -f "ssh.*kreata.ee" 2>/dev/null || true

   # Wait a moment for ports to be released
   sleep 1

   # Step 1: Set up SSH reverse proxy to expose over HTTPS
   # This creates https://localdev.kreata.ee → localhost:7003
   ssh -R 7003:localhost:7003 kreata.ee -f -N

   # Step 2: Clear Redis database (start fresh)
   redis-cli -n 5 FLUSHDB

   # Step 3: Start EmailEngine from source (foreground process)
   # No need to change directories - run from documentation folder

   # Set environment variables as needed:
   # - EE_OPENAPI_VERBOSE=true - Verbose OpenAPI logging
   # - EENGINE_DISABLE_SETUP_WARNINGS=true - No admin password warnings
   # - EENGINE_REQUIRE_API_AUTH=false - Allow API calls without tokens
   # - EENGINE_PREPARED_LICENSE - License key (REQUIRED for IMAP workers to start)
   # - EENGINE_SETTINGS - JSON configuration including serviceUrl
   # - Add other environment variables as needed for specific features

   EE_OPENAPI_VERBOSE=true \
   EENGINE_DISABLE_SETUP_WARNINGS=true \
   EENGINE_REQUIRE_API_AUTH=false \
   EENGINE_PREPARED_LICENSE="$(cat /Users/andris/Downloads/license-B841200C688A23F553E0DC45.txt)" \
   EENGINE_SETTINGS='{"serviceUrl":"https://localdev.kreata.ee","webhooks":"https://kreata.ee/s","webhookEvents":["*"],"webhooksEnabled":true}' \
   node /Users/andris/Projects/emailengine/server.js \
     --dbs.redis='redis://127.0.0.1:6379/5' \
     --api.port=7003 \
     --api.host=0.0.0.0

   # EmailEngine will start and display logs in the terminal
   # Wait a few seconds for it to fully initialize
   # Once you see "Server started" or similar, it's ready for testing/screenshots
   # Leave this terminal running - use a new terminal for other commands
   ```

   **Important - EmailEngine Process Behavior:**

   - EmailEngine runs as a **foreground process**, not a background daemon
   - Logs are printed directly to stdout in the terminal
   - Keep the terminal running while testing or taking screenshots
   - Use a separate terminal window for running screenshot scripts or API calls
   - Stop EmailEngine with `Ctrl+C` when done
   - Wait 3-5 seconds after startup before accessing the UI or API

   **Key Points:**

   - Access to latest unreleased features and source code
   - OAuth2 credentials available in `/Users/andris/Projects/emailengine/.env`
   - HTTPS access via SSH tunnel required for OAuth2 callbacks
   - Webhook testing with `https://kreata.ee/s` webhook sink
   - Full control over environment and debugging
   - Always clear Redis database before starting to ensure clean state
   - Port 7003 avoids conflict with Docusaurus (port 3000)

   **Screenshot Capture Scripts:**

   - Location: `scripts/` directory
   - Tool: Playwright (Chromium browser automation)
   - Viewport size: **1600x900** (standard resolution for documentation screenshots)
   - Browser: Chromium in headless mode with `--ignore-certificate-errors` flag

   **Available Scripts:**

   - `scripts/capture-screenshots.js` - Basic screenshot capture without authentication

     - Captures 10 main UI pages (dashboard, accounts, settings, webhooks, templates, etc.)
     - Uses `fullPage: false` for viewport screenshots, `fullPage: true` for scrollable pages
     - Target URL configured in script (default: `https://localdev.kreata.ee`)

   - `scripts/capture-api-examples.js` - API response and webhook payload screenshots

     - Creates syntax-highlighted code examples for API responses
     - Makes actual API calls to capture real responses
     - Renders JSON responses as styled code blocks using highlight.js
     - GitHub Dark theme for consistent appearance

   - `scripts/capture-collapse-screenshots.sh` - quoted-thread collapse control for `docs/receiving/web-safe-html.md`
     - Self-contained: boots a throwaway EmailEngine from a local checkout (`EE_REPO`, default `../emailengine`) on the isolated e2e Redis db, then tears it down. Never touches a real install
     - Stages the conversation it photographs: provisions an Ethereal mailbox, uploads an original message and a Gmail-shaped reply that quotes it as raw RFC822, then opens the reply in the admin message browser
     - Borrows Playwright, nodemailer and the Ethereal/bootstrap helpers from the EmailEngine checkout, so this repo needs no extra dependencies
     - Writes `web-safe-html-collapsed.png` and `web-safe-html-expanded.png`
     - Message bodies are constants at the top of `capture-collapse-screenshots.js` - edit those to photograph a different client's quoting style
     - `EE_PORT=7098` if something already holds 7099, `EE_HEADED=1` to watch it, `EE_KEEP_RUNNING=1` to leave the instance up

   - `scripts/capture-mcp-screenshots.js` - the MCP admin screenshots for `docs/mcp/*`
     - Needs `EE_URL` and `EE_PASSWORD` (every shot requires an authenticated admin session: minting a token and approving a client both refuse without one). `EE_ACCOUNT` fills the "limit to one account" fields, default `docs-demo-account`
     - Turns `mcpEnabled` and `mcpOAuthEnabled` ON, mints mcp-scoped tokens and registers a dynamic OAuth client, so point it at a throwaway instance
     - Deletes existing mcp-scoped tokens first, so re-runs do not stack duplicate rows in the tokens listing
     - The generated token is redacted in the DOM before the shot, so `mcp-connect-generated.png` never carries a live-looking credential
     - Writes `mcp-settings`, `mcp-connect-token`, `mcp-connect-generated`, `mcp-connect-oauth`, `mcp-tools-catalog`, `mcp-oauth-consent`, `mcp-token-form` and `mcp-tokens-list`

   **Output Directories:**

   - UI screenshots: `static/img/screenshots/`
   - API examples: `static/img/examples/`

   **Screenshot Timing:**

   - Uses `waitUntil: 'networkidle'` or `'domcontentloaded'` for page load
   - Adds `waitForTimeout(1000-3000ms)` for dynamic content to render
   - Longer waits for Bull Board queues (2000ms) due to animation/loading

   **Running Screenshot Capture:**

   ```bash
   # Basic capture (no authentication needed if EENGINE_REQUIRE_API_AUTH=false)
   node scripts/capture-screenshots.js

   # API examples capture
   node scripts/capture-api-examples.js

   # Quoted-thread collapse control (boots and tears down its own EmailEngine)
   ./scripts/capture-collapse-screenshots.sh

   # MCP admin screenshots (docs/mcp/*)
   EE_URL=http://127.0.0.1:7003 EE_PASSWORD=... node scripts/capture-mcp-screenshots.js
   ```

   **Testing Resources:**

   **Ethereal Email (Test IMAP/SMTP Accounts)**

   Create temporary test email accounts for development and screenshots:

   ```bash
   # Create a new Ethereal test account
   curl -X POST https://api.nodemailer.com/user \
     -H "Content-type: application/json" \
     -d '{
       "requestor": "emailengine-dev",
       "version": "0.0.1"
     }'
   ```

   Example response:

   ```json
   {
     "status": "success",
     "user": "kua5fisxkbcwqe6a@ethereal.email",
     "pass": "CKrU9TCXK6Zn1N3NkF",
     "smtp": {
       "host": "smtp.ethereal.email",
       "port": 587,
       "secure": false
     },
     "imap": {
       "host": "imap.ethereal.email",
       "port": 993,
       "secure": true
     },
     "pop3": {
       "host": "pop3.ethereal.email",
       "port": 995,
       "secure": true
     },
     "web": "https://ethereal.email"
   }
   ```

   **Important Ethereal Email Behavior:**

   - Does NOT send real emails to external addresses
   - Emails sent via SMTP are added to the sender's INBOX instead
   - Perfect for testing: sending triggers both `messageSent` and `messageNew` webhooks
   - No email delivery to real recipients - safe for testing

   **OAuth2 Credentials**

   Gmail OAuth2 credentials are stored in:

   - `/Users/andris/Projects/emailengine/.env`
   - Use these credentials for testing OAuth2 account setup
   - Required for OAuth2 flow screenshots

   **Webhook Testing**

   Use the webhook helper service for testing webhook delivery:

   ```bash
   # Read latest webhooks (newest first)
   curl https://kreata.ee/r

   # Read N newest webhooks
   curl https://kreata.ee/r?lines=10

   # Clear all webhooks
   curl https://kreata.ee/r/clear

   # Configure EmailEngine to send webhooks to this sink
   # Set webhook URL to: https://kreata.ee/s
   ```

   **Special Cases:**

   - **Authentication Form Screenshot (03-account-add-form.png):**

     - Cannot use `/admin/accounts/new` URL (doesn't exist)
     - Must use the "Generate authentication link" API endpoint: `POST /v1/authentication/form`
     - **Important:** Set `EENGINE_SETTINGS='{"serviceUrl":"https://localdev.kreata.ee"}'` when starting EmailEngine, otherwise the form URLs will be invalid
     - Example process:

       ```bash
       # 1. Generate authentication form link for IMAP
       curl -X POST "https://localdev.kreata.ee/v1/authentication/form" \
         -H "Content-Type: application/json" \
         -d '{
           "account": "test-imap",
           "type": "imap",
           "name": "Test Account",
           "redirectUrl": "https://localdev.kreata.ee/admin/accounts"
         }'

       # 2. Response includes the form URL: {"url": "https://localdev.kreata.ee/accounts/new?data=..."}
       # 3. Open the URL in Playwright with certificate error ignoring
       # 4. Capture screenshot at 1600x900 resolution with fullPage: true
       ```

     - The authentication form API creates a temporary hosted form
     - `redirectUrl` is a required field - user is redirected here after authentication
     - Use `type: "imap"` to show IMAP/SMTP configuration fields
     - Use `type: "oauth2"` with `provider` to show OAuth2 flow
     - Form URL is time-limited and single-use

   **Notes:**

   - Screenshots are automatically overwritten if they exist
   - Failed captures show warnings but don't stop the script
   - Browser closes automatically after completion
   - Uses HTTPS with certificate error ignoring for local development
   - All scripts create output directories if they don't exist
   - Update script URLs to point to `http://localhost:3030` when using Docker container

7. **Dependency Vulnerabilities Do Not Apply Here** - ignore every alert, and DO NOT upgrade React, marked, or related packages

   This site is published as a **static site** - dependency vulnerabilities reported by GitHub/npm have no runtime impact since no server-side code runs in production. Never report a Dependabot count as something to look at, raise it as a finding, or offer to fix it: for this repository the answer is always no.

   **DO NOT upgrade these packages:**
   - `react` / `react-dom` - Must stay on v18.x (OpenAPI theme incompatible with React 19)
   - `marked` - Current version required for OpenAPI docs plugin compatibility
   - `docusaurus-theme-openapi-docs` - Tightly coupled with specific React/marked versions
   - Other OpenAPI-related dependencies

   **Why:**
   - The `docusaurus-plugin-openapi-docs` and `docusaurus-theme-openapi-docs` packages have strict peer dependency requirements
   - Upgrading React to v19 breaks the OpenAPI theme rendering
   - Upgrading marked breaks markdown processing in API docs
   - These vulnerabilities are false positives for static sites - the vulnerable code paths are never executed

   **If Dependabot/GitHub reports vulnerabilities:**
   - This repository builds and deploys a **static site only** - the generated HTML, CSS, and JS in `build/` is the sole deployed artifact. There is no server-side runtime, no database, and no untrusted input processed in production. Every dependency lives in the build/dev toolchain and never runs in the deployed site.
   - Therefore treat **all** Dependabot and dependency security alerts for this repository as **not relevant**. They are not exploitable in production. Dismiss them in GitHub (Security > Dependabot) with reason "Vulnerable code is not actually used" (`not_used`).
   - Do not upgrade dependencies solely to clear such alerts - especially react, marked, and OpenAPI-related packages, which breaks the build (see above).
   - Test thoroughly with `npm run build` after any dependency change you do make for other reasons.

8. **Never Hardcode the License Price** - Use the `Price` component

   The price changes and is region-dependent, so no markdown file may state a figure.

   - Usage: add `import Price from '@site/src/components/Price';` after the frontmatter, then append `<Price />` to an authored pricing sentence, with no space before it:
     ```markdown
     - **Annual license:** Flat annual fee<Price />, excluding VAT - see [postalsys.com/plans](https://postalsys.com/plans)
     ```
   - It renders a parenthetical carrying the current price, or nothing at all. The authored sentence is never modified, so it must read correctly on its own
   - The figure and currency come from `https://postalsys.com/region.js` (`window.PSYS_REGION`), the same source emailengine.app reads: EUR for EU visitors, USD elsewhere
   - Nothing is rendered during SSR, so the static build and the Algolia index stay free of a number that can go stale. That guarantee depends on the Algolia crawler not executing JavaScript, and its config lives in the Algolia dashboard rather than in this repo, so enabling `renderJavaScript` there would index a live price with no local signal
   - Find every page using it with `grep -rl '<Price' docs/`
   - The figure arrives from region.js already formatted; the plausibility band and the currency symbols live in `postalsys-web`, and a price failing the band means the key is simply absent. Never reintroduce a price range or a symbol map here
   - Verify with `npm run verify-pricing` after deploying. It checks the live sites, not your working tree, so run it once the deploy has landed, and it needs a browser (`npx playwright install chromium`). It also covers the three emailengine.app pages that state a price, each of which carries its own verbatim copy of that site's inline currency script, and is the only check anywhere that would catch a syntax error in one of them

## Quick Commands

```bash
# Development server (hot reload)
npm start                    # → http://localhost:3000

# Production build (auto-updates API docs from emailengine.dev)
npm run build                # Output: ./build/

# Test production build locally
npm run serve                # → http://localhost:3000

# Update OpenAPI spec and regenerate API docs + sidebar
npm run update-swagger       # Downloads latest, regenerates docs & sidebar

# Generate API docs from OpenAPI spec
npm run docusaurus gen-api-docs all

# Generate API sidebar structure (outputs to console)
npm run generate-api-sidebar

# Clear cache (if build issues)
npm run docusaurus clear
```

## Architecture Overview

### Documentation Structure

This is a **unified documentation system** where each feature/topic is covered by a single comprehensive document that merges:

- API documentation (from OpenAPI spec)
- Blog post content (practical tutorials, examples)
- General documentation (concepts, configuration)

**DO NOT** create separate docs for the same topic. Always enhance the existing unified documentation.

```
docs/
├── index.md                 # Landing page
├── getting-started/         # 2 files - Introduction, quick start
├── installation/            # 6 files - Platform-specific install guides
├── accounts/                # 19 files - Gmail, Outlook, OAuth2, service accounts
├── sending/                 # 13 files - Basic sending, mail merge, threading, templates
├── receiving/               # 10 files - Messages, searching, attachments, web safe HTML
├── webhooks/                # 26 files - Overview, routing, per-event payload docs
├── mcp/                     # 5 files  - MCP endpoint for AI agents: overview, connecting, tools, access control, protocol
├── configuration/           # 10 files - Environment variables, Redis, settings
├── api-reference/           # 7 files - Overview, tokens, accounts, messages, sending, webhooks, OpenAPI
├── integrations/            # 5 files - PHP, CRM, AI/ChatGPT, low-code
├── advanced/                # 14 files - Performance, monitoring, encryption, IDs
├── deployment/              # 7 files - Docker, SystemD, Render, Nginx, security
├── reference/               # 6 files - Webhook events, error codes, config options, llm-context.md
├── troubleshooting/         # 1 file  - Common problems and fixes
├── licensing/               # 1 file  - License and privacy
├── comparison/              # 2 files - EmailEngine vs Nylas, vs Unipile
├── support/                 # 2 files - Support channels, security FAQ
└── api/                     # 80 auto-generated OpenAPI docs (DO NOT EDIT)

static/
├── capabilities.json        # Machine-readable API capabilities (for AI agents)
└── img/                     # Images and screenshots
```

### Key Architectural Decisions

1. **Blog Disabled** - All blog content has been elevated to unified documentation

   - `docusaurus.config.ts` has `blog: false`
   - Blog posts were merged into topic-based docs (e.g., OAuth2 setup, mail merge)

2. **Auto-Generated Sidebars** - Structure comes from the file tree

   - Sidebar structure comes from file organization and frontmatter
   - `sidebar_position` in frontmatter controls order
   - A subdirectory has no frontmatter of its own, so its label and position come from a `_category_.json`. Three exist for that reason; nothing else needs one

3. **OpenAPI Integration** - API docs auto-generated from `sources/swagger.json`

   - Plugin: `docusaurus-plugin-openapi-docs`
   - Output: `docs/api/` (81 endpoint files, plus `emailengine-api.info.mdx`)
   - Configuration: See `docusaurus.config.ts` plugins section
   - "Send API Request" button disabled (`hideSendButton: true`) since EmailEngine is self-hosted
   - Server URL automatically replaced with `https://emailengine.example.com`

4. **Single Source of Truth** - Each topic has ONE authoritative page
   - No duplicate content between docs/blog/API
   - Cross-references use relative links

## Download Links

EmailEngine provides shortened download URLs that redirect to the latest GitHub releases:

| File                          | Short URL (Latest)                             | Versioned URL Format                                            | Full GitHub URL                                                                       |
| ----------------------------- | ---------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **macOS PKG (Intel)**         | https://go.emailengine.app/emailengine.pkg     | https://go.emailengine.app/download/v2.79.4/emailengine.pkg     | https://github.com/postalsys/emailengine/releases/latest/download/emailengine.pkg     |
| **macOS PKG (Apple Silicon)** | https://go.emailengine.app/emailengine-arm.pkg | https://go.emailengine.app/download/v2.79.4/emailengine-arm.pkg | https://github.com/postalsys/emailengine/releases/latest/download/emailengine-arm.pkg |
| **Linux Binary (tar.gz)**     | https://go.emailengine.app/emailengine.tar.gz  | https://go.emailengine.app/download/v2.79.4/emailengine.tar.gz  | https://github.com/postalsys/emailengine/releases/latest/download/emailengine.tar.gz  |
| **Source Distribution**       | https://go.emailengine.app/source-dist.tar.gz  | https://go.emailengine.app/download/v2.79.4/source-dist.tar.gz  | https://github.com/postalsys/emailengine/releases/latest/download/source-dist.tar.gz  |
| **Windows Executable**        | https://go.emailengine.app/emailengine.exe     | https://go.emailengine.app/download/v2.79.4/emailengine.exe     | https://github.com/postalsys/emailengine/releases/latest/download/emailengine.exe     |

**Download URL Formats:**

1. **Latest version (recommended for most docs):**

   - Format: `https://go.emailengine.app/<filename>`
   - Example: `https://go.emailengine.app/emailengine.exe`
   - Always downloads the latest release

2. **Specific version (when version pinning is needed):**
   - Format: `https://go.emailengine.app/download/vX.X.X/<filename>`
   - Example: `https://go.emailengine.app/download/v2.79.4/emailengine.exe`
   - Downloads a specific release version

**Important Notes:**

- **PKG installers (macOS only):** Architecture-specific binaries
  - `emailengine.pkg` - Intel x86
  - `emailengine-arm.pkg` - Apple Silicon (M1 and newer)
- **Binary (Linux only):** `emailengine.tar.gz` - Linux x86 only (no ARM binary available)
- **Source distribution:** `source-dist.tar.gz` - Platform-independent, requires Node.js 20+ (per `engines` in the EmailEngine package.json; 24+ recommended), works on any platform
- **Windows:** `emailengine.exe` - Windows executable (NOT `emailengine-win-x64.exe`)
- Always use the short URLs in documentation (they're easier to remember and maintain)
- The installer script is available at: https://go.emailengine.app (redirects to install.sh)

## Important Files

### Configuration Files

- **`docusaurus.config.ts`** - Main Docusaurus configuration

  - Site metadata (title, URL, organization)
  - OpenAPI plugin configuration (sources/swagger.json → docs/api/)
  - Navigation (navbar, footer)
  - Theme settings (dark mode, syntax highlighting)
  - Blog disabled: `blog: false`

- **`sidebars.ts`** - Sidebar configuration

  - `docsSidebar`: Auto-generated from docs/ structure
  - `apiSidebar`: Auto-generated from OpenAPI spec

- **`package.json`** - Dependencies and scripts
  - Docusaurus 3.9.2
  - OpenAPI plugin and theme
  - Build, start, serve scripts

### Source Files

The `sources/` directory contains original reference materials used to create the unified documentation. These files are kept for historical reference and should NOT be edited directly.

**Directory Structure:**

- **`sources/swagger.json`** - EmailEngine OpenAPI 3.0.0 specification (auto-updated on build)

  - Published spec: 60 paths, 83 operations, all under `/v1`. After `update-swagger` strips the excluded Document Store tag, the local copy has 58 paths and 81 operations
  - Auto-downloaded from https://go.emailengine.app/swagger.json during build
  - Used to generate `docs/api/` content (81 endpoint files)
  - Documented for end users in `docs/api-reference/openapi-spec.md`
  - Run `npm run update-swagger` to update from production

- **`sources/openapi/`** - OpenAPI schema definitions

  - Contains API schema files and specifications
  - Used as reference for API documentation structure
  - DO NOT edit - historical reference only

- **`sources/blog/`** - Original blog articles (Ghost CMS format)

  - 43+ blog posts covering tutorials and detailed guides
  - Topics include OAuth2 setup, mail merge, encryption, etc.
  - Content has been merged into unified topic-based docs
  - DO NOT edit - historical reference only

- **`sources/website-md/`** - Old documentation website (Markdown)
  - 33 HTML/Markdown files from previous documentation site
  - General documentation covering features and configuration
  - Content has been unified into current `docs/` structure
  - DO NOT edit - historical reference only

**Important Notes:**

- All content from these sources has been merged into the unified `docs/` directory
- When updating documentation, edit files in `docs/`, not in `sources/`
- The sources are kept for reference and to track content origin
- Only `sources/swagger.json` is actively updated (automatically during build)

**Reference Locations:**

- EmailEngine source code: `/Users/andris/Projects/emailengine`
- API schema definitions: `./sources/openapi/`
- Blog articles: `./sources/blog/`
- Old documentation: `./sources/website-md/`

## EmailEngine Source Code Reference

The EmailEngine application source code is located at `/Users/andris/Projects/emailengine` and can be used as a reference when writing documentation.

### Source Code Structure

```
/Users/andris/Projects/emailengine/
├── server.js                 # Main server entry point (110KB - core application)
├── package.json              # Application metadata (v2.79.4)
├── lib/                      # Core library modules
│   ├── account.js           # Account management logic (106KB)
│   ├── schemas.js           # API validation schemas (78KB)
│   ├── routes-ui.js         # Web UI routes (382KB)
│   ├── tools.js             # Utility functions (74KB)
│   ├── oauth2-apps.js       # OAuth2 application handling (49KB)
│   ├── webhooks.js          # Webhook management (21KB)
│   ├── get-raw-email.js     # Email retrieval logic (18KB)
│   ├── autodetect-imap-settings.js  # IMAP autodiscovery (23KB)
│   ├── bounce-detect.js     # Bounce detection (25KB)
│   ├── arf-detect.js        # ARF (Abuse Report Format) detection
│   ├── es.js                # Elasticsearch integration (19KB)
│   ├── gateway.js           # SMTP gateway functionality
│   ├── templates.js         # Email template management
│   ├── threads.js           # Email threading logic
│   ├── tokens.js            # API token management (10KB)
│   ├── settings.js          # Application settings (9KB)
│   ├── metrics-collector.js # Metrics and monitoring (7KB)
│   ├── encrypt.js           # Encryption utilities (5KB)
│   ├── db.js                # Database abstraction (9KB)
│   ├── logger.js            # Logging utilities
│   ├── consts.js            # Application constants (8KB)
│   ├── api-routes/          # API endpoint route handlers
│   ├── email-client/        # Email client implementations
│   ├── oauth/               # OAuth2 authentication modules
│   ├── imapproxy/           # IMAP proxy functionality
│   └── lua/                 # Redis Lua scripts
├── workers/                  # Background worker processes
│   ├── api.js               # API worker (362KB - main API logic)
│   ├── documents.js         # Document indexing worker (45KB)
│   ├── imap.js              # IMAP sync worker (32KB)
│   ├── smtp.js              # SMTP submission worker (18KB)
│   ├── submit.js            # Email submission logic (15KB)
│   ├── webhooks.js          # Webhook delivery worker (20KB)
│   └── imap-proxy.js        # IMAP proxy worker
├── views/                    # Handlebars templates for web UI
├── static/                   # Static assets (CSS, JS, images)
├── config/                   # Configuration files
├── test/                     # Test suite
├── scripts/                  # Build and deployment scripts
├── systemd/                  # SystemD service files
├── examples/                 # Example configurations and code
├── data/                     # Runtime data directory
└── translations/             # i18n translation files
```

### Key Source Files for Documentation Reference

**Core Application:**

- `server.js` - Main entry point, server initialization
- `lib/account.js` - Account creation, management, authentication
- `lib/schemas.js` - Complete API request/response schemas (use for API documentation)
- `workers/api.js` - Main API endpoint implementations

**Email Operations:**

- `lib/get-raw-email.js` - Email fetching and processing
- `workers/imap.js` - IMAP synchronization logic
- `workers/smtp.js` - SMTP submission handling
- `lib/threads.js` - Email threading algorithm
- `lib/bounce-detect.js` - Bounce detection and handling

**Authentication & Security:**

- `lib/oauth2-apps.js` - OAuth2 application management (Gmail, Outlook)
- `lib/oauth/` - OAuth2 provider implementations
- `lib/encrypt.js` - Encryption utilities
- `lib/tokens.js` - API token management

**Configuration & Settings:**

- `lib/settings.js` - Application settings management
- `lib/autodetect-imap-settings.js` - IMAP server autodiscovery
- `config/` - Default configuration files

**Integration & Extensions:**

- `lib/webhooks.js` - Webhook configuration and management
- `workers/webhooks.js` - Webhook delivery implementation
- `lib/gateway.js` - SMTP gateway functionality
- `lib/templates.js` - Email template system
- `lib/es.js` - Elasticsearch integration

**Utilities:**

- `lib/tools.js` - General utility functions
- `lib/db.js` - Database/Redis abstraction
- `lib/logger.js` - Logging infrastructure
- `lib/metrics-collector.js` - Metrics and monitoring

### Using Source Code as Reference

When documenting EmailEngine features:

1. **Check implementation details** - Review the actual source code to understand how features work
2. **Find examples** - Look in `examples/` directory for real-world usage patterns
3. **Verify API schemas** - Use `lib/schemas.js` for accurate API documentation
4. **Understand OAuth2 flows** - Reference `lib/oauth2-apps.js` and `lib/oauth/` for OAuth2 setup
5. **Worker architecture** - Review `workers/` directory to understand background processing
6. **Configuration options** - Check `lib/settings.js` and `config/` for all available settings
7. **Test cases** - Review `test/` directory for usage examples and edge cases

### Important Notes

- The source code is actively developed - check version in `package.json` (currently v2.79.4)
- OpenAPI spec is generated from this codebase - available at https://go.emailengine.app/swagger.json
- Web UI templates in `views/` use Handlebars templating
- Background workers use Bull queues (BullMQ) for job processing
- Redis is the primary data store - Lua scripts in `lib/lua/` for atomic operations

## Documentation Authoring Guidelines

These are the mechanics. The editorial stance behind them is the MDN model described at the top of this file, which is what settles anything the mechanics do not cover.

### Conventions the whole site follows

Settled during the site-wide review on 25 August 2026. Match them rather than reopening them:

- **Host placeholder:** `https://emailengine.example.com` on any page addressing a running deployment. `http://localhost:3000` only where the reader has just started an instance locally, and never both in one page without a sentence saying why.
- **Examples must run.** No pseudo-HTTP in a `javascript` fence, no `//` or `#` comments inside a `json` block, no `{account}` left in a curl URL, no `...` standing in for a value. NDJSON samples are fenced as `text`, because they are several documents rather than one.
- **No prices, ours or anyone else's.** Ours goes through the `Price` component. A competitor's or a host's gets named as a model, with a link to their page and a date.
- **No counts that drift.** Do not write "all 81 endpoints"; link to the reference instead.
- **Every page ends with `## See Also`,** four or five links that each say what the reader would go there for.
- **Verify before writing.** The spec in `sources/swagger.json` settles request and response shapes; `/Users/andris/Projects/emailengine` settles behavior. A plausible-sounding claim that neither supports does not go in.

### File Structure

Every documentation file should have:

```markdown
---
title: Clear, Descriptive Title
sidebar_position: 1
description: Brief one-sentence description for SEO
---

# Page Title

<!-- Source attribution (for reference) -->
<!-- Sources: docs/old-file.md, blog/tutorial.md, api/endpoint.md -->

Brief introduction paragraph.

:::tip Quick Example
Practical example or key takeaway
:::

## Section 1

Content with code examples...

## See Also

- [Related Topic 1](/docs/category/file1)
- [Related Topic 2](/docs/category/file2)
```

### Code Examples

Always provide **multi-language examples** where appropriate:

```markdown
**cURL:**
\`\`\`bash
curl -X POST http://localhost:3000/v1/account
\`\`\`

**Node.js:**
\`\`\`javascript
const response = await fetch('http://localhost:3000/v1/account')
\`\`\`

**Python:**
\`\`\`python
response = requests.post('http://localhost:3000/v1/account')
\`\`\`

**PHP:**
\`\`\`php
$response = $client->post('http://localhost:3000/v1/account');
\`\`\`
```

### Cross-References

Use relative links to other documentation:

```markdown
- [Account Setup](/docs/accounts) - Link to section
- [Gmail OAuth2](/docs/accounts/gmail-imap) - Link to specific page
- [API Reference](/docs/api-reference) - Link to API overview
- [Webhooks API](/docs/api/webhooks) - Link to auto-generated API doc
```

### Admonitions

Use Docusaurus admonitions for important information:

```markdown
:::tip Best Practice
Use OAuth2 for Gmail and Outlook in production
:::

:::warning Security
Never commit OAuth2 credentials to version control
:::

:::danger Breaking Change
Version 3.0 removes legacy authentication
:::

:::info Note
This feature requires a license key
:::
```

## Common Tasks

### Adding New Documentation

1. **Identify the topic** - Check existing docs first
2. **Choose the right section** - accounts/, sending/, receiving/, etc.
3. **Create the file** with proper frontmatter
4. **Use existing files as templates** - Maintain consistent structure
5. **Test the build** - `npm run build`

### Updating API Documentation

The API documentation is automatically updated from the EmailEngine production API whenever you run a build.

#### Automatic Updates (Recommended)

When you run `npm run build`, the following happens automatically:

1. **Downloads latest OpenAPI spec** from https://go.emailengine.app/swagger.json
2. **Replaces server URL** - Changes `http://0.0.0.0:6677` to `https://emailengine.example.com` (since EmailEngine is self-hosted)
3. **Regenerates API docs** from the updated spec (81 endpoint files)
4. **Regenerates API sidebar** with collapsible tag-based groups (18 categories)
5. **Builds the site** with the latest API documentation

This is handled by the `prebuild` script that runs `scripts/update-swagger.js`.

**Note:** The server URL is automatically replaced because EmailEngine is self-hosted software. The documentation uses `https://emailengine.example.com` as a placeholder that users should replace with their actual server URL.

#### Manual Updates

If you need to update the API docs without building:

```bash
# Update swagger.json and regenerate everything
npm run update-swagger

# Or do it step by step:
# 1. Download latest spec manually or edit sources/swagger.json
# 2. Regenerate API docs
npm run docusaurus gen-api-docs all
# 3. Regenerate API sidebar structure
npm run generate-api-sidebar
# 4. Review and commit the updated sidebars.ts file
```

**Important:** The `generate-api-sidebar` script outputs the sidebar structure to the console. You must manually copy this into `sidebars.ts` to update the `apiSidebar` array.

#### API Sidebar Structure

The API sidebar is organized by OpenAPI tags into 18 collapsible categories:

- Account (13 endpoints)
- Mailbox (4 endpoints)
- Message (10 endpoints)
- Submit, Outbox, Delivery Test, Access Tokens, Settings, Templates, Logs, Stats, License, Webhooks, OAuth2 Applications, SMTP Gateway, Blocklists, Multi Message Actions, Export (Beta)

The sidebar structure is manually maintained in `sidebars.ts` for full control over organization and ordering.

### Fixing Build Warnings

Most warnings are **non-critical** (broken anchors from old structure):

```bash
# Clear cache and rebuild
npm run docusaurus clear
npm run build

# Check specific warnings
npm run build 2>&1 | grep "Broken link"
```

### Deploying

```bash
# 1. Verify build passes
npm run build

# 2. Test production build
npm run serve

# 3. Deploy (depends on hosting platform)
# - Render: Auto-deploy from GitHub
# - GitHub Pages: npm run deploy
# - Custom: Upload ./build/ directory
```

## Key Concepts

### EmailEngine Features Covered

- **Account Management** - Gmail (OAuth2, API, service accounts), Outlook (OAuth2), generic IMAP/SMTP
- **Sending Emails** - Basic sending, replies, mail merge, threading, templates, queue management
- **Receiving Emails** - Webhooks, message operations, searching, attachments, tracking
- **Configuration** - Environment variables, Redis, prepared settings
- **Integrations** - PHP SDK, CRM integration, AI/ChatGPT, low-code platforms, Cloudflare Workers
- **Advanced Topics** - Performance tuning, monitoring, logging, encryption, ID system
- **Deployment** - Docker, SystemD, Render, Nginx reverse proxy, security hardening
- **API Reference** - Complete API documentation with authentication, examples, error handling

### Documentation Quality Standards

Every page should:

- ✅ Have complete frontmatter (title, sidebar_position, description)
- ✅ Include practical code examples
- ✅ Provide step-by-step procedures where applicable
- ✅ Cross-reference related documentation
- ✅ Include troubleshooting tips
- ✅ Follow consistent formatting and tone
- ✅ Use admonitions for important information
- ✅ Provide "See Also" section with related links

### What NOT to Do

- ❌ **Don't create duplicate documentation** - Enhance existing unified docs instead
- ❌ **Don't edit `docs/api/` directly** - These are auto-generated from OpenAPI spec
- ❌ **Don't re-enable the blog** - Blog content is now part of unified docs
- ❌ **Don't add `_category_.json` files for ordering pages** - Page order comes from `sidebar_position` in frontmatter. The three that exist (`docs/accounts/gmail/`, `docs/accounts/microsoft-365/`, `docs/sending/threading/`) label and position a *subdirectory*, which frontmatter cannot do. Add one only for that.
- ❌ **Don't create separate "tutorial" or "guide" sections** - Merge into relevant topic docs
- ❌ **Don't commit with broken builds** - Always run `npm run build` first
- ❌ **Don't use emojis in documentation** - Use text instead (see Important Rules section)
- ❌ **Don't add AI attribution to git commits** - Keep commit messages professional (see Important Rules section)

## Troubleshooting

### Build Fails

```bash
# Clear cache
npm run docusaurus clear
rm -rf .docusaurus build node_modules/.cache

# Reinstall dependencies
npm install

# Rebuild
npm run build
```

### Port Already in Use

```bash
# Kill existing server
lsof -ti:3000 | xargs kill -9

# Or use different port
npm start -- --port 3001
```

### OpenAPI Generation Issues

```bash
# Check OpenAPI spec is valid
npm run docusaurus gen-api-docs:version:emailengine

# Force regeneration
rm -rf docs/api
npm run docusaurus gen-api-docs all
```

## Project History

This documentation was created by:

1. **Setting up Docusaurus 3.9.1** with TypeScript and OpenAPI plugin
2. **Migrating 33 HTML docs** from EmailEngine website to Markdown
3. **Converting 43 blog posts** from Ghost CMS format to Docusaurus
4. **Generating 73 API docs** from OpenAPI specification
5. **Unifying 149 source files** into 71 comprehensive topic-based documentation files
6. **Removing old content** (blog, old docs, helper scripts) to create single source of truth

**Result of that initial unification:** 100% feature coverage across 71 topic-based files. The site has grown since - see Project Status at the top for current figures.

## Getting Help

- **Docusaurus docs:** https://docusaurus.io/docs
- **OpenAPI plugin:** https://github.com/PaloAltoNetworks/docusaurus-openapi-docs
- **EmailEngine:** https://emailengine.app

---

**Last Updated:** August 26, 2026
**Docusaurus Version:** 3.9.2
**EmailEngine API Version:** 2.79.4
**Status:** Production Ready
