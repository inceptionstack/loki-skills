---
name: playwright-cli
description: "Browser automation via playwright-cli. Navigate pages, click elements, fill forms, take screenshots, inspect network traffic, and manage cookies/storage. Use when: (1) testing web UIs end-to-end, (2) scraping or extracting data from web pages, (3) taking screenshots or PDFs, (4) filling forms or clicking buttons programmatically, (5) debugging network requests, (6) automating login flows with saved state. NOT for: API-only testing (use curl), static file reading, or non-browser tasks."
---

# playwright-cli — Browser Automation

Direct CLI for headless browser automation. No MCP protocol needed — call via bash.

## Setup

```bash
npm install -g @playwright/cli
playwright-cli install          # downloads Chromium (~186MB, one-time)
```

## Core Workflow

Every session follows: **open → interact → close**

```bash
playwright-cli open "https://example.com"     # launch browser, navigate
playwright-cli snapshot                        # get page structure (refs)
playwright-cli click e5                        # click element by ref
playwright-cli fill e8 "hello world"           # fill input by ref
playwright-cli screenshot                      # save viewport PNG
playwright-cli close                           # close browser
```

## Element Targeting

`snapshot` returns an accessibility tree with `[ref=eN]` identifiers.
Use these refs in subsequent commands:

```bash
playwright-cli snapshot
# Output:
# - textbox "Search" [ref=e3]
# - button "Submit" [ref=e4]
# - link "About" [ref=e5]

playwright-cli click e5          # click the "About" link
playwright-cli fill e3 "query"   # type into search box
```

## Key Commands

### Navigation
```bash
playwright-cli open "https://url.com"    # open browser + navigate
playwright-cli goto "https://other.com"  # navigate existing session
playwright-cli go-back                   # browser back
playwright-cli go-forward                # browser forward
playwright-cli reload                    # reload page
```

### Interaction
```bash
playwright-cli click <ref>               # click element
playwright-cli dblclick <ref>            # double-click
playwright-cli fill <ref> "text"         # clear + type into input
playwright-cli type "text"               # type into focused element
playwright-cli select <ref> "value"      # select dropdown option
playwright-cli check <ref>               # check checkbox
playwright-cli uncheck <ref>             # uncheck checkbox
playwright-cli hover <ref>               # hover over element
playwright-cli press Enter               # press keyboard key
playwright-cli upload file.pdf           # upload file
```

### Inspection
```bash
playwright-cli snapshot                  # full page accessibility tree
playwright-cli snapshot e5               # subtree of specific element
playwright-cli screenshot                # viewport screenshot (PNG)
playwright-cli screenshot e5             # element screenshot
playwright-cli pdf                       # save page as PDF
playwright-cli console                   # show console messages
```

### Network
```bash
playwright-cli requests                  # list all network requests
playwright-cli request 3                 # full details of request #3
playwright-cli response-body 3           # response body of request #3
playwright-cli route "**/*.jpg"          # mock/block matching URLs
```

### Tabs
```bash
playwright-cli tab-list                  # list open tabs
playwright-cli tab-new "https://..."     # open new tab
playwright-cli tab-select 1              # switch to tab
playwright-cli tab-close 2               # close tab
```

### Storage & Auth
```bash
playwright-cli state-save auth.json      # save cookies + localStorage
playwright-cli state-load auth.json      # restore saved session
playwright-cli cookie-list               # list all cookies
playwright-cli cookie-set name value     # set a cookie
playwright-cli localstorage-list         # list localStorage
```

## Sessions

Multiple browser sessions can run in parallel:

```bash
playwright-cli -s=admin open "https://admin.example.com"
playwright-cli -s=user open "https://app.example.com"
playwright-cli -s=admin snapshot        # inspect admin session
playwright-cli -s=user click e4         # interact with user session
playwright-cli list                      # show all sessions
playwright-cli close-all                 # close everything
```

## Output Formats

```bash
playwright-cli snapshot --json           # structured JSON output
playwright-cli requests --json           # JSON network log
playwright-cli --raw snapshot            # raw value only (no status)
```

## Tips

- Always `snapshot` after navigation or interaction to get fresh refs
- Refs change when the page re-renders — don't cache them across actions
- Use `--json` for programmatic parsing of results
- `screenshot` saves to `.playwright-cli/` directory by default
- For login flows: `state-save` after authenticating, `state-load` to reuse
- `eval "document.title"` runs arbitrary JS and returns the result
- `resize 1920 1080` sets viewport before screenshots for consistent sizing
