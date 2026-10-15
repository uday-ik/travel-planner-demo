# Accessible Travel Planner: a web page for an n8n workflow

A single web page that sends a trip request to an **n8n workflow** through its **webhook** and shows the checked plan that comes back. The page is only the front door. The team of AI agents (Coordinator, Flight Agent, Itinerary Agent, Hotel Agent, Constraint Checker) runs inside n8n.

Public page: https://uday-ik.github.io/travel-planner-demo/

```
Browser (this page)  --POST {"message": "..."}-->  n8n webhook  -->  5 agents  -->  {"output": "...checked plan..."}  --> page
```

**This page contains no keys and no webhook URL.** Each person types their own webhook URL into the page. Nothing works until they have their own published n8n workflow.

Tested on Windows 11 with Chrome. The Mac steps use the same commands in Terminal and have **not been run on a Mac yet**, so check them once on a Mac before you rely on them.

---

## Quick start: choose your path

Either way, you make the changes in **your own n8n**. The page cannot do anything until you have a workflow with a webhook that is published.

| | Path 1: use the public page | Path 2: have your own copy of the page |
|---|---|---|
| **You use** | https://uday-ik.github.io/travel-planner-demo/ | Your own page, on GitHub Pages or on your own computer |
| **In n8n** | Add the webhook, publish, copy the Production URL, and allow `https://uday-ik.github.io` as the origin | The same, but allow *your* page's address as the origin |
| **On GitHub** | Nothing | Fork or clone this repository (and turn on GitHub Pages if you want it online) |
| **Good for** | Trying it quickly, or showing it to a group with one shared link | Changing the page, or running it without depending on someone else's link |

What you always do in n8n (sections 3 to 6):
1. Add a **Webhook** node and connect it to the first agent.
2. **Publish** the workflow and copy the **Production URL**.
3. Set **Allowed Origins (CORS)** to the address of the page you use.
4. Paste the Production URL into the page.

---

## Contents
1. [What you need](#1-what-you-need)
2. [Three ways to run the page](#2-three-ways-to-run-the-page)
3. [Step 1: Add a webhook to your n8n workflow](#3-step-1-add-a-webhook-to-your-n8n-workflow)
4. [Step 2: Publish and get the Production URL](#4-step-2-publish-and-get-the-production-url)
5. [Step 3: Test the webhook from the terminal](#5-step-3-test-the-webhook-from-the-terminal)
6. [Step 4: Set CORS (Allowed Origins)](#6-step-4-set-cors-allowed-origins)
7. [Step 5: Use the page](#7-step-5-use-the-page)
8. [Run the page on your own computer (Windows and Mac)](#8-run-the-page-on-your-own-computer-windows-and-mac)
9. [Put your own copy on GitHub Pages](#9-put-your-own-copy-on-github-pages)
10. [Safety and misuse: questions and answers](#10-safety-and-misuse-questions-and-answers)
11. [Troubleshooting](#11-troubleshooting)
12. [Test prompts and expected answers](#12-test-prompts-and-expected-answers)
13. [How it works under the hood](#13-how-it-works-under-the-hood)

---

## 1. What you need
- The **Week 3 Part 1 workflow** from the course (the Accessible Travel Planning Team in n8n), built and working in your own n8n account, with its credentials and the two Google Sheets.
- An **n8n account**. n8n Cloud works. A self-hosted n8n works too, if it has a public address.
- A modern browser (Chrome is recommended).
- A terminal: **PowerShell** on Windows, **Terminal** on Mac.
- Optional, for running the page locally: **Git** and **Python 3**.

## 2. Three ways to run the page
You only need one. The only thing that changes is the address you put in the Allowed Origins setting in step 4.

| Way | Page address | Allowed Origins value in n8n |
|---|---|---|
| **A. Use the public page** (nothing to install) | `https://uday-ik.github.io/travel-planner-demo/` | `https://uday-ik.github.io` |
| **B. Your own copy on GitHub Pages** | `https://YOURNAME.github.io/travel-planner-demo/` | `https://YOURNAME.github.io` |
| **C. On your own computer** | `http://localhost:8000` | `http://localhost:8000` |

Way A works for everyone. The page keeps no data: each person pastes their **own** webhook URL, and the page sends the request from their browser to their own n8n. Allowed Origins is set in each person's n8n, so each person's n8n only has to allow the page's address.

For a quick first test you can set Allowed Origins to `*` (allow every site). Change it to the specific address afterwards.

## 3. Step 1: Add a webhook to your n8n workflow
Your workflow already starts from a **Chat Trigger** (for the chat in the n8n editor). You add a second front door, a **Webhook**, so other programs and web pages can call it. The same team answers both.

1. Open the workflow in n8n.
2. Click **+** and add a **Webhook** node. Set:
   - **HTTP Method:** `POST`
   - **Path:** `travel-planner` (see [section 10](#10-safety-and-misuse-questions-and-answers) for why a harder-to-guess path is better)
   - **Authentication:** None (for the demo)
   - **Respond:** `When Last Node Finishes`
   - **Response Data:** `First Entry JSON`
3. Connect the Webhook node's output to the **Coordinator** node. Leave the Chat Trigger connected too.
4. Open the **Coordinator** node. Change **Source for Prompt (User Message)** to **Define below**. In the **Prompt** box, enter this expression:

   ```
   {{ $json.chatInput || $json.body.message }}
   ```

   It reads the message from the chat, or from the webhook, whichever started the run.
5. Save the workflow.

## 4. Step 2: Publish and get the Production URL
1. Click **Publish** at the top right of the editor. Older versions call this the **Active** toggle. The webhook only exists while the workflow is published. If you skip this you will get `404 ... webhook is not registered`.
2. Double-click the Webhook node. It shows two URLs:
   - **Test URL** (contains `/webhook-test/`). It only works while you click **Listen for test event**. Not for this page.
   - **Production URL** (contains `/webhook/`). This is the one you use.
3. Copy the **Production URL**. It looks like this:

   ```
   https://YOUR-NAME.app.n8n.cloud/webhook/travel-planner
   ```

Production calls do not appear on the canvas. Open the **Executions** tab to see them.

## 5. Step 3: Test the webhook from the terminal
Do this before using the page. If the terminal works, any problem later is about the page or CORS, not the workflow.

### Windows (PowerShell)
Paste these two lines. Use **single quotes** so `$900` is not read as a variable. Replace the URL with yours.

```powershell
$body = @{ message = 'Plan a trip from SFO to Lisbon on Oct 12, 2026. 3 nights, total budget $900. I need a direct flight, wheelchair assistance, and a hotel with a wheelchair-accessible room and step-free entrance.' } | ConvertTo-Json
$r = Invoke-RestMethod -Uri 'PASTE_PRODUCTION_URL' -Method Post -ContentType 'application/json' -Body $body
$r.output
```

Copy only the commands, not the `PS C:\...>` prompt or any red error text from an earlier run.

### Mac (Terminal)
Paste this. It uses `curl`, which Macs have built in. Keep the single quotes. Replace the URL with yours.

```bash
curl -s -X POST 'PASTE_PRODUCTION_URL' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Plan a trip from SFO to Lisbon on Oct 12, 2026. 3 nights, total budget $900. I need a direct flight, wheelchair assistance, and a hotel with a wheelchair-accessible room and step-free entrance."}'
```

To print only the plan, add this at the end of the command, on the same line as the last quote:

```bash
 | python3 -c "import sys, json; print(json.load(sys.stdin)['output'])"
```

### What you should see
After 20 to 40 seconds (the agents are working), a plan with `FLIGHT FL101`, `HOTEL HT201`, `TOTAL $830` and `OVERALL_VERDICT: APPROVED`.

On Windows, `curl` in PowerShell is a different command (an alias). Use `Invoke-RestMethod` as above, or type `curl.exe`.

## 6. Step 4: Set CORS (Allowed Origins)
**Why:** a web page may only call another website if that website says it is allowed. This setting tells n8n which page addresses may call your webhook from a browser. Without it, the terminal works but the page says "The request did not reach n8n".

1. Double-click the **Webhook** node.
2. Open **Options** (click **Add Option** if you do not see it) and choose **Allowed Origins (CORS)**.
3. Enter the address from the table in [section 2](#2-three-ways-to-run-the-page). Use the **address only**: no path, no slash at the end.
   - Public page: `https://uday-ik.github.io`
   - Your GitHub Pages: `https://YOURNAME.github.io`
   - Local computer: `http://localhost:8000`
4. To allow more than one, separate them with commas, for example `https://uday-ik.github.io,http://localhost:8000` (check that your n8n version accepts a comma-separated list).
5. **Save** the workflow, then **Publish** again. The change only takes effect after you publish.

For a first test, `*` allows any site. Use `*` if you double-click `index.html` from your computer. That counts as an unusual origin.

Optional check from the terminal (shows whether n8n sends the permission header):

```bash
# Mac
curl -s -i -X POST 'PASTE_PRODUCTION_URL' -H 'Content-Type: application/json' -H 'Origin: https://uday-ik.github.io' -d '{"message":"Plan a trip to Paris."}' | head -15
```

```powershell
# Windows
$r = Invoke-WebRequest -UseBasicParsing -Uri 'PASTE_PRODUCTION_URL' -Method Post -ContentType 'application/json' -Headers @{ Origin = 'https://uday-ik.github.io' } -Body (@{ message = 'Plan a trip to Paris.' } | ConvertTo-Json)
$r.Headers['Access-Control-Allow-Origin']
```

Look for `access-control-allow-origin` in the answer. If it shows your address (or `*`), CORS is set. This check is a guide, not a guarantee, because browsers also send a "preflight" request first. The real test is the page itself.

## 7. Step 5: Use the page
1. Open the page (public, your copy, or local).
2. Paste your **Production URL** into the first box. The page keeps it in your own browser only.
3. Click one of the preset buttons or type a request, then click **Plan my trip**.
4. Wait 20 to 40 seconds. You will see a status line with the time taken, a green **APPROVED** or red **NEEDS REVISION** badge, and the full plan.

## 8. Run the page on your own computer (Windows and Mac)
Use this for way C. A local web server is more reliable than double-clicking the file.

### Get the files
Install Git if you do not have it (Windows: https://git-scm.com. Mac: type `git --version` in Terminal, and macOS offers to install the developer tools).

```bash
git clone https://github.com/uday-ik/travel-planner-demo.git
cd travel-planner-demo
```

The same two commands work in PowerShell on Windows and in Terminal on Mac. If you do not want to use Git, download the repo as a ZIP from GitHub (green **Code** button, then **Download ZIP**) and unzip it.

### Start a local web server

| | Command |
|---|---|
| **Windows (PowerShell)** | `python -m http.server 8000` (or `py -m http.server 8000`) |
| **Mac (Terminal)** | `python3 -m http.server 8000` |

Leave the terminal window open. Then open **http://localhost:8000** in your browser. Set Allowed Origins to `http://localhost:8000`. To stop the server, press **Ctrl+C** in the terminal.

If Python is not installed: Windows https://www.python.org/downloads/ (tick "Add Python to PATH"). Mac: `python3 --version` should work after the developer tools install; if not, use https://www.python.org/downloads/.

### The quick alternative: open the file directly
- **Windows:** double-click `index.html`, or run `start index.html`.
- **Mac:** double-click `index.html`, or run `open index.html`.

This works only if Allowed Origins is `*`.

## 9. Put your own copy on GitHub Pages
Use this for way B.

1. On GitHub, open this repository and click **Fork** (top right), or create a new **public** repository and upload `index.html`. GitHub Pages is free for public repositories.
2. In your copy, open **Settings > Pages**.
3. Under **Build and deployment**, set **Source** to **Deploy from a branch**, **Branch** to **main**, **Folder** to **/ (root)**, and click **Save**.
4. After about a minute, refresh. Your page is at `https://YOURNAME.github.io/REPO-NAME/`.
5. In n8n, set Allowed Origins to `https://YOURNAME.github.io` and publish again.

To change the page, edit `index.html` in your copy and commit. GitHub rebuilds the page in about a minute.

## 10. Safety and misuse: questions and answers

**Can anyone misuse this because the repository is public?**
Not through the repository. It holds no keys, no passwords and no webhook URL. A stranger who opens the page sees an empty URL box and nothing to call.

**So where is the real risk?**
In your **webhook URL**. Anyone who knows it can call it, and each call runs the agents and uses your model API key. That includes someone using PowerShell or `curl`: Allowed Origins (CORS) only controls which *web pages* may call it from a browser. It does not stop direct calls.

**How do I reduce that risk?**
1. **Do not share the URL.** Do not put it in screenshots, recordings or a public file. Anyone who watches a screen share can copy it.
2. **Keep it published only while you need it.** Switch **Publish** off after the demo. The URL then stops working (it answers 404).
3. **Set a spending limit** on the model API key you use (in the OpenAI, OpenRouter or Anthropic account). This is your strongest protection.
4. **Use a harder-to-guess Path**, for example `travel-planner-k7x92m` instead of `travel-planner`. Change it again after a demo.
5. **For apps that call n8n directly,** turn on **Header Auth** on the Webhook node, with a secret that only those apps know. Do not use this for the browser page, because a secret in a public page is no longer secret. This guide has not tested Header Auth with the page.

**Someone else wants to use this project. How does that work?**
They need their own n8n workflow with its own webhook. Then they either use the public page (and allow `https://uday-ik.github.io` in their n8n), or fork it and run their own copy. They never use your webhook, and they cannot, because they do not have your URL.

**Why is my Allowed Origins value my GitHub link?**
Because it names the one website allowed to call your webhook from a browser. It does not stop anyone from opening that same page. They would just be sending requests to *their own* webhook.

**I want to clone the repository and run it on my own computer. What do I do?**
Follow [section 8](#8-run-the-page-on-your-own-computer-windows-and-mac). Set Allowed Origins to `http://localhost:8000`, or to `https://uday-ik.github.io` if you use the public page. Publish your workflow only while you are using it, and switch it off afterwards.

## 11. Troubleshooting

| What you see | Likely cause and fix |
|---|---|
| `404 ... webhook "POST travel-planner" is not registered` | The workflow is not published, or you used the Test URL. Publish it and use the Production URL. |
| Page says "The request did not reach n8n" | CORS (Allowed Origins is missing or has a typo), a wrong URL, or the workflow is not published. Check all three. |
| Works with `*` but not with the specific address | Check for a trailing slash, a path, `http` instead of `https`, or a missing re-publish. If it still fails, use `*` for the demo and rely on the safety steps in section 10. |
| Terminal works, page does not | CORS. The terminal does not use CORS. |
| Page works, but the answer is empty or says it has no message | The Coordinator is not reading the webhook message. Check the expression in step 1, item 4. |
| `curl` on Windows does something strange | In PowerShell `curl` is an alias. Use `Invoke-RestMethod` or `curl.exe`. |
| Mac says `zsh: command not found: python` | Use `python3`. |
| `Address already in use` when starting the local server | Another program uses port 8000. Use `python3 -m http.server 8001` and allow `http://localhost:8001`. |
| Takes about 30 seconds | Normal. Five agents are working. |
| Works in the n8n editor but fails when published | A credential or setting is missing. Open the failed run in **Executions**. |
| The model API says quota or billing | Add credit or check the key in your provider account. |

## 12. Test prompts and expected answers

| Test | Request | Expected |
|---|---|---|
| Happy path | SFO to Lisbon, Oct 12, 2026, 3 nights, budget $900, direct flight, wheelchair assistance, accessible hotel | FL101 + HT201 = **$830, APPROVED** |
| Other date | Same, Oct 13 | FL105 + HT201 = **$890, APPROVED** |
| Budget fails | Oct 12, 4 nights, budget $800 | **$970, NEEDS REVISION** |
| No flight fits | Oct 12, 3 nights, budget $300 | **NO_MATCHING_FLIGHT** |
| Wrong city | SFO to Paris | "No data available for this destination..." |

The page has a button for each of these.

## 13. How it works under the hood

**Request** (the page sends this):

```
POST https://YOUR-NAME.app.n8n.cloud/webhook/travel-planner
Content-Type: application/json

{ "message": "Plan a trip from SFO to Lisbon on Oct 12, 2026. ..." }
```

**Response** (n8n sends this when the last agent finishes):

```
200 OK
{ "output": "ITINERARY_UNDER_REVIEW: ... OVERALL_VERDICT: APPROVED" }
```

- **API:** a set of addresses a program offers so that other programs can ask it for things. The webhook is one of these addresses, so your workflow is now a small API.
- **Webhook:** an address that waits for requests.
- **CORS:** a browser rule. A page may only call another site if that site names the page's address as allowed.
- **Status codes:** `200` worked, `404` wrong address or workflow not published, `500` the server failed.

The page sends the request with `fetch` and shows the `output` text as plain text. It never inserts the answer as HTML.
