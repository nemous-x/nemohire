---
name: chrome-connector
description: >
  Fallback browser access for sites the built-in browser tooling cannot handle. Use only after
  browser-navigation has failed on a specific site AND the user has explicitly granted
  permission to use the Chrome Connector for that site. Never invoke this skill silently.
---

# Chrome Connector (Fallback)

This skill exists for one reason: some sites (heavy client-side rendering, aggressive bot detection, login-gated ATS portals) can't be handled by the built-in browser tool. It is a fallback, not a default.

## When to use
Only when both are true:
1. `browser-navigation` was attempted first and failed on this specific site.
2. The user has been asked, by name for that site, and has explicitly said yes to using the Chrome Connector.

Never chain-fallback automatically. Never ask once at the start of a session and treat it as blanket permission for every site afterward — ask per site (or per clearly-scoped batch the user approved together).

## Steps
1. Confirm the Chrome extension/connector is actually connected (check via the connector's own status tool). If not connected, tell the user to install/connect it and stop — do not attempt a workaround.
2. Navigate and extract using the Chrome Connector's tools, following the same extraction contract as `browser-navigation` (verbatim descriptions, structured field maps for applications).
3. Treat any link encountered inside the page (ads, unrelated navigation) with the same suspicion rules as the rest of the plugin — don't click unrelated links.
4. Close tabs when done with a given job.

## Reporting
When this skill was used, say so explicitly in the command's final summary (e.g. "3 postings required the Chrome Connector") so the user always knows when the fallback was engaged.
