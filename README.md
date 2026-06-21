<div align="center">

# 🌉 Bulk SharePoint MCP

### One MCP server. Two AI brains. Every SharePoint list — in plain English.

**The first SharePoint [Model Context Protocol](https://modelcontextprotocol.io) server that connects _natively_ to both Microsoft Copilot Studio and Claude Desktop — and filters lists past the 5,000-item wall with natural language.**

_No Power Automate flows. No custom connector. No threshold errors._

<br>

![Status](https://img.shields.io/badge/status-production-3ddc84?style=for-the-badge)
![MCP](https://img.shields.io/badge/protocol-MCP-16c2d5?style=for-the-badge)
![Azure](https://img.shields.io/badge/Azure-Container_Apps-0a1f2e?style=for-the-badge&logo=microsoftazure)
![Graph](https://img.shields.io/badge/Microsoft-Graph-ff8a3d?style=for-the-badge&logo=microsoft)

![Python](https://img.shields.io/badge/python-3.12-blue?logo=python&logoColor=white)
![FastMCP](https://img.shields.io/badge/FastMCP-3.2.4-16c2d5)
![OAuth](https://img.shields.io/badge/auth-OAuth_2.0_·_OBO-ff8a3d)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

</div>

---

## ⚡ The 30-second pitch

You open a SharePoint list with **47,000 rows** and ask one question:

> _"How many open high-priority tickets came from EMEA this week?"_

SharePoint normally throws **`This view cannot be displayed because it exceeds the list view threshold`** and you reach for an Excel export or a three-day Power Automate build.

With this server, you just **ask your AI** — and get the answer in three seconds. The same server answers **Copilot Studio** at work and **Claude Desktop** on your laptop.

```text
You      ▸  how many open high-priority EMEA tickets this week?
Agent    ▸  Connected to Support (47,212 items). 318 open high-priority
            EMEA tickets this week. Top driver: Payments API (94).
            Want a CSV or a breakdown by assignee?
```

---

## 🧠 Why this is different

| | Traditional approach | **Bulk SharePoint MCP** |
|---|---|---|
| Big lists (5,000+) | ❌ Threshold error | ✅ Indexed filter + pagination |
| Power Automate flow | ❌ Required | ✅ None |
| Custom connector | ❌ Required for Copilot | ✅ None — Dynamic discovery |
| Works with Copilot Studio | ⚠️ Painful | ✅ Native |
| Works with Claude Desktop | ❌ Usually not | ✅ Native |
| Query language | OData gymnastics | 🗣️ **Plain English** |
| One identity for everything | ❌ | ✅ One Entra app |

---

## 🏗️ Architecture

```mermaid
flowchart TD
    A["🟦 Copilot Studio<br/>Dynamic discovery (DCR)"]:::client
    B["🟧 Claude Desktop<br/>Client registration (DCR)"]:::client
    A -->|OAuth 2.0 · Entra ID · v2 tokens| M
    B -->|OAuth 2.0 · Entra ID · v2 tokens| M
    M["🔌 <b>Bulk SharePoint MCP Server</b><br/>17 tools · FastMCP · Azure Container Apps"]:::hub
    M -->|On-Behalf-Of token exchange| G
    G["📊 Microsoft Graph"]:::graph
    G --> S["📁 SharePoint Online<br/>5,000 / 50,000 / 500,000 rows — same"]:::spo

    classDef client fill:#0c2430,stroke:#16c2d5,color:#eaf6f8;
    classDef hub fill:#0a1f2e,stroke:#3ddc84,stroke-width:2px,color:#fff;
    classDef graph fill:#0c2430,stroke:#16c2d5,color:#eaf6f8;
    classDef spo fill:#0c2430,stroke:#ff8a3d,color:#eaf6f8;
```

**One server, two front doors, the same identity.** Both AI clients register dynamically and receive a token scoped to the server's API. The server then performs an **On-Behalf-Of exchange** for a real Microsoft Graph token — so _your_ permissions flow all the way through to SharePoint.

---

## 🧱 Breaking the 5,000-item wall

SharePoint throttles any query that would scan more than 5,000 rows. The only queries that survive filter or sort on an **indexed column**. The server is built entirely around that fact:

1. **Read the index map** — `list_schema` reports which columns are indexed.
2. **Filter on indexed columns** — natural language → OData `$filter` the index can satisfy.
3. **Page through everything** — results stream via Graph's `@odata.nextLink` in safe batches.

> 💡 **The insight:** don't fight the threshold — _feed the index_. Teach the AI which columns are indexed, turn plain English into index-friendly filters, and the wall stops mattering.

---

## 🚀 Quick start

> **Prerequisites:** an Azure subscription, the [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli), PowerShell, and a SharePoint list.

```powershell
# 1. Sign in
az login

# 2. Deploy everything — Entra app, container, secrets, the lot
cd bulk-sp-mcp
./deploy.ps1
```

The script provisions the **entire** Entra App Registration automatically (identity URI, `user_impersonation` scope, v2 tokens, Graph permissions, admin consent, client secret), builds the container in ACR, wires secrets and CORS, and verifies health.

✅ **Success looks like:**
```text
>>> 4. Verify the new image is actually live
  OK Health check: version 6.1 live
```

<details>
<summary><b>🔐 What the script builds in Entra (click to expand)</b></summary>

| Step | What | Why |
|---|---|---|
| A | App registration (single tenant) | The server's identity |
| B | Expose API + `user_impersonation` scope | Token audience the server recognizes |
| C | `requestedAccessTokenVersion = 2` | FastMCP requires v2 tokens |
| D | Graph delegated perms + admin consent | `Sites.ReadWrite.All`, `User.Read`, `offline_access`… |
| E | Redirect URIs | Where sign-in returns |
| F | Client secret | For the On-Behalf-Of exchange |

You don't click any of this — `deploy.ps1` does it via `az` in ~20 seconds.
</details>

---

## 🔌 Connect your AI

<details open>
<summary><b>Microsoft Copilot Studio</b></summary>

<br>

1. Agent → **Tools** → **Add a tool** → **New tool** → **Model Context Protocol**
2. **Server URL:** `https://<your-app>.azurecontainerapps.io/mcp`
3. **Authentication:** `OAuth 2.0` → **Dynamic discovery** ← _the whole trick_
4. **Create** → **Create new connection** → sign in → tools load automatically

</details>

<details>
<summary><b>Claude Desktop</b></summary>

<br>

**Settings → Connectors → Add custom connector →** paste the `/mcp` URL → **Connect** → sign in.

Or in `claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "bulk-sharepoint": {
      "type": "http",
      "url": "https://<your-app>.azurecontainerapps.io/mcp"
    }
  }
}
```
</details>

---

## 🧰 The 17 tools

| | Tool | What it does |
|---|---|---|
| 🔗 | `connect_list` | URL → live list + index map |
| 📋 | `list_connected_lists` | Lists connected this session |
| 🗂️ | `list_schema` | Columns, types, and what's indexed |
| 🔄 | `refresh_cache` | Re-pull a list's index map |
| ✂️ | `disconnect_list` | Drop a list from the session |
| 🎯 | `get_record_by_id` | One item by id (never throttled) |
| 🔎 | `search_records` | Keyword search |
| 🧮 | `filter_records` | `eq` / `ne` / `gt` / `lt` / `contains` / `startswith` |
| 📊 | `aggregate_records` | count / sum / avg / min / max + group_by |
| 🕰️ | `find_stale_items` | Untouched in N days |
| 👯 | `detect_duplicates` | Same value across many rows |
| 📤 | `export_to_csv` | Filtered rows → CSV |
| ➕ | `create_item` | Add a row |
| 📦 | `bulk_create_tasks` | Add many rows |
| ✏️ | `update_item` | Edit a row |
| 🗑️ | `delete_item` | Recycle a row |
| 🪪 | `whoami` | Who's signed in (connectivity test) |

---

## 🩹 Lessons learned the hard way

_This didn't work on the first try. Or the tenth. Here's what broke — so you skip the pain._

<details>
<summary><b>Server crash-looped on boot</b></summary>
A newer framework release removed a method the server used. <b>Fix:</b> pin <code>fastmcp[azure]==3.2.4</code> and avoid the removed kwarg.
</details>

<details>
<summary><b>Worked in Claude Desktop, failed in Copilot Studio</b></summary>
A custom connector routes OAuth through a different proxy → endless token errors. <b>Fix:</b> use Copilot's <b>Dynamic discovery</b> — the same DCR path Claude already uses. No connector.
</details>

<details>
<summary><b><code>InvalidAuthenticationToken — Invalid audience</code> reaching Graph</b></summary>
A token can't carry two audiences. The user's API token isn't a Graph token. <b>Fix:</b> the server now does an MSAL <b>On-Behalf-Of</b> exchange for a real Graph token.
</details>

<details>
<summary><b>Deploy crashed with a <code>charmap</code> UnicodeEncodeError</b></summary>
The ACR build log contains characters the Windows console can't print. <b>Fix:</b> force UTF-8 (<code>chcp 65001</code>) and build with <code>az acr build --no-logs</code> instead of streaming.
</details>

<details>
<summary><b><code>Client Not Registered</code> after a redeploy</b></summary>
Each new revision resets the in-memory client registry. <b>Fix:</b> delete the connection in Copilot Studio and create a new one — it re-registers instantly.
</details>

---

## 🛠️ Tech stack

**Azure Container Apps** · **FastMCP** (Streamable HTTP) · **Microsoft Graph** · **Microsoft Entra ID** (OAuth 2.0, DCR, On-Behalf-Of) · **Python 3.12** · **MSAL**

---

## 📁 Repo layout

```text
bulk-sp-mcp/
├── server.py                       # The MCP server — 17 tools + health routes
├── deploy.ps1                      # One-shot Azure provisioning (every fix baked in)
├── requirements.txt                # Pinned deps
├── Dockerfile                      # python:3.12-slim, port 8080
├── COPILOT-STUDIO-SETUP.md         # Dynamic discovery walkthrough
├── claude_desktop_config.json      # Claude Desktop connector entry
└── connector-*.json                # Custom connector (fallback only)
```

---

## 🗺️ Roadmap

- [ ] Persistent client registry (survive redeploys)
- [ ] Multi-list joins in one query
- [ ] Saved natural-language views
- [ ] Webhook triggers on list changes

---

## 👤 Author

**Kerolos** — Senior Infrastructure &amp; Security Engineer
_Building the bridges that were supposed to be impossible — then writing down how._

**Power of Automation** · [Blog](https://powerofautomation2025.blogspot.com)

> Open to conversations with teams building the future of agents on the Microsoft stack. 🚀

---

<div align="center">

**If this saved you a three-day flow build, drop a ⭐**

_Built for beginners, tested in production._

</div>
