# Deploy an n8n workflow on the web

This project shows how to take an **n8n workflow** and put it on the web: first as an **API** that any program can call, then behind a **web page** hosted for free on **GitHub Pages**.

The worked example is an *Accessible Travel Planning Team*: five AI agents in n8n that read a trip request and return a checked plan. The web page in this repository sends the request and shows the answer.

Public page: https://uday-ik.github.io/travel-planner-demo/

```
Web page  --POST {"message": "..."}-->  n8n webhook  -->  your workflow  -->  {"output": "..."}  -->  Web page
```

## What you will learn
- How a **webhook** turns an n8n workflow into an API.
- What a request and a response look like (URL, method, headers, JSON body, status codes).
- Why a browser needs **CORS** permission, and how to set it in n8n.
- How to host a front end for free on **GitHub Pages**.
- How to keep a public endpoint from being misused.

The page has no keys and no webhook URL inside it. Each person types the URL of their own webhook into the page.

---

## Quick start: choose your path

Either way, you make the changes in **your own n8n**. The page does nothing until you have a workflow with a published webhook.

| | Path 1: use the public page | Path 2: have your own copy of the page |
|---|---|---|
| **You use** | https://uday-ik.github.io/travel-planner-demo/ | Your own page, on GitHub Pages or on your own computer |
| **In n8n** | Add the webhook, publish, copy the Production URL, allow `https://uday-ik.github.io` as the origin | The same, but allow *your* page's address as the origin |
| **On GitHub** | Nothing | Fork or clone this repository, and turn on GitHub Pages if you want it online |
| **Good for** | Trying it quickly with one shared link | Changing the page, or not depending on someone else's link |

What you always do in n8n (sections 3 to 6):
1. Add a **Webhook** node and connect it to the first node of your workflow.
2. **Publish** the workflow and copy the **Production URL**.
3. Set **Allowed Origins (CORS)** to the address of the page you use.
4. Paste the Production URL into the page.

---

## Contents
1. [What you need](#1-what-you-need)
2. [Three ways to run the page](#2-three-ways-to-run-the-page)
3. [Step 1: Add a webhook to your workflow](#3-step-1-add-a-webhook-to-your-workflow)
4. [Step 2: Publish and get the Production URL](#4-step-2-publish-and-get-the-production-url)
5. [Step 3: Test the webhook from the terminal](#5-step-3-test-the-webhook-from-the-terminal)
6. [Step 4: Set CORS (Allowed Origins)](#6-step-4-set-cors-allowed-origins)
7. [Step 5: Use the page](#7-step-5-use-the-page)
8. [Run the page on your own computer](#8-run-the-page-on-your-own-computer)
9. [Put your own copy on GitHub Pages](#9-put-your-own-copy-on-github-pages)
10. [Concepts](#10-concepts)
11. [Safety](#11-safety)
12. [Troubleshooting](#12-troubleshooting)
13. [Test prompts for the example workflow](#13-test-prompts-for-the-example-workflow)

---

## 1. What you need
- An **n8n** account. n8n Cloud works. A self-hosted n8n works too, if it has a public address.
- A workflow to put online. The page is built around the travel planning example, but any workflow that takes some text and returns text works.
- A browser. Chrome is recommended.
- A terminal: **PowerShell** on Windows, **Terminal** on Mac.
- Optional, for running the page on your own computer: **Git** and **Python 3**.

## 2. Three ways to run the page
You only need one. The only thing that changes is the address you enter in Allowed Origins in step 4.

| Way | Page address | Allowed Origins value in n8n |
|---|---|---|
| **A. The public page** (nothing to install) | `https://uday-ik.github.io/travel-planner-demo/` | `https://uday-ik.github.io` |
| **B. Your own copy on GitHub Pages** | `https://YOURNAME.github.io/travel-planner-demo/` | `https://YOURNAME.github.io` |
| **C. On your own computer** | `http://localhost:8000` | `http://localhost:8000` |

Way A works for anyone. The page stores nothing: each person pastes their **own** webhook URL, and the request goes from their browser to their own n8n. Allowed Origins is set in each person's n8n, so it only needs to allow the page's address.

For a first test, `*` allows every site. Change it to the specific address afterwards.

## 3. Step 1: Add a webhook to your workflow
A webhook is a second way into your workflow. Other programs and web pages can call it.

1. Open your workflow in n8n.
2. Click **+** and add a **Webhook** node. Set:
   - **HTTP Method:** `POST`
   - **Path:** `travel-planner` (see [Safety](#11-safety) for why a harder-to-guess path is better)
   - **Authentication:** None for a first test
   - **Respond:** `When Last Node Finishes`
   - **Response Data:** `First Entry JSON`
3. Connect the Webhook node's output to the **first node** of your workflow. Any existing trigger can stay connected.
4. Make the first node read the message from the webhook. In the example, the first node is an AI Agent called **Coordinator**:
   - Open it and set **Source for Prompt (User Message)** to **Define below**.
   - In the **Prompt** box, enter:

   ```
   {{ $json.chatInput || $json.body.message }}
   ```

   This reads the message from the chat trigger or from the webhook, whichever started the run. In a workflow that is not built around a chat, use `{{ $json.body.message }}` wherever the workflow expects its input.
5. Save the workflow.

The webhook returns whatever the **last node** outputs. For an AI Agent that is `{"output": "..."}`. The page shows the `output`, `text` or `message` field of the answer, or the raw text if there is none.

## 4. Step 2: Publish and get the Production URL
1. Click **Publish** at the top right of the editor. Older versions call this the **Active** toggle. The webhook only exists while the workflow is published. Without it you get `404 ... webhook is not registered`.
2. Double-click the Webhook node. It shows two URLs:
   - **Test URL** (contains `/webhook-test/`). It only works while you click **Listen for test event**. Do not use it for the page.
   - **Production URL** (contains `/webhook/`). This is the one you use.
3. Copy the **Production URL**:

   ```
   https://YOUR-NAME.app.n8n.cloud/webhook/travel-planner
   ```

Production calls do not show on the canvas. Open the **Executions** tab to see them.

## 5. Step 3: Test the webhook from the terminal
Do this before using the page. If the terminal works, any later problem is about the page or CORS, not the workflow.

### Windows (PowerShell)
Paste these lines. Use **single quotes**, so `$900` is not read as a variable. Replace the URL with yours.

```powershell
$body = @{ message = 'Plan a trip from SFO to Lisbon on Oct 12, 2026. 3 nights, total budget $900. I need a direct flight, wheelchair assistance, and a hotel with a wheelchair-accessible room and step-free entrance.' } | ConvertTo-Json
$r = Invoke-RestMethod -Uri 'PASTE_PRODUCTION_URL' -Method Post -ContentType 'application/json' -Body $body
$r.output
```

Copy only the commands, not the `PS C:\...>` prompt or any red error text from an earlier run. In PowerShell, `curl` is an alias for another command, so use `Invoke-RestMethod` as above, or type `curl.exe`.

### Mac (Terminal)
Macs have `curl` built in. Keep the single quotes. Replace the URL with yours.

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
After a few seconds to about half a minute (several agents run in turn), a plan with `FLIGHT FL101`, `HOTEL HT201`, `TOTAL $830` and `OVERALL_VERDICT: APPROVED`.

## 6. Step 4: Set CORS (Allowed Origins)
**Why:** a web page may only call another site if that site says it is allowed. This setting tells n8n which page addresses may call your webhook from a browser. Without it, the terminal works but the page says "The request did not reach n8n".

1. Double-click the **Webhook** node.
2. Open **Options** (click **Add Option** if you do not see it) and choose **Allowed Origins (CORS)**.
3. Enter the address from the table in [section 2](#2-three-ways-to-run-the-page). Use the **address only**: no path, and no slash at the end.
   - Public page: `https://uday-ik.github.io`
   - Your GitHub Pages: `https://YOURNAME.github.io`
   - Your own computer: `http://localhost:8000`
4. **Save** the workflow, then **Publish** again. The change only takes effect after you publish.

If you open `index.html` by double-clicking it, the page has no normal address, so use `*` for that.

Optional check from the terminal. It shows whether n8n sends the permission header for an origin. Look for `access-control-allow-origin` in the answer:

```bash
# Mac
curl -s -i -X POST 'PASTE_PRODUCTION_URL' -H 'Content-Type: application/json' -H 'Origin: https://uday-ik.github.io' -d '{"message":"Plan a trip to Paris."}' | head -15
```

```powershell
# Windows
$r = Invoke-WebRequest -UseBasicParsing -Uri 'PASTE_PRODUCTION_URL' -Method Post -ContentType 'application/json' -Headers @{ Origin = 'https://uday-ik.github.io' } -Body (@{ message = 'Plan a trip to Paris.' } | ConvertTo-Json)
$r.Headers['Access-Control-Allow-Origin']
```

The final check is the page itself.

## 7. Step 5: Use the page
1. Open the page (public, your own copy, or local).
2. Paste your **Production URL** into the first box. The page keeps it in your own browser only.
3. Click a preset button or type a request, then click **Plan my trip**.
4. Wait for the answer. You see a status line with the time taken, a green **APPROVED** or red **NEEDS REVISION** badge, and the full plan.

The preset buttons are examples for the travel workflow. For another workflow, type your own text in the box.

## 8. Run the page on your own computer
A local web server is more reliable than opening the file directly.

### Get the files
Install Git if you do not have it (Windows: https://git-scm.com. Mac: type `git --version` in Terminal, and macOS offers to install the developer tools).

```bash
git clone https://github.com/uday-ik/travel-planner-demo.git
cd travel-planner-demo
```

The same two commands work in PowerShell and in Terminal. Without Git, use the green **Code** button on GitHub, then **Download ZIP**, and unzip it.

### Start a local web server

| | Command |
|---|---|
| **Windows (PowerShell)** | `python -m http.server 8000` (or `py -m http.server 8000`) |
| **Mac (Terminal)** | `python3 -m http.server 8000` |

Leave the terminal window open and open **http://localhost:8000** in your browser. Set Allowed Origins to `http://localhost:8000`. Press **Ctrl+C** in the terminal to stop the server.

If Python is not installed, get it from https://www.python.org/downloads/ (on Windows, tick "Add Python to PATH").

### Open the file directly
- **Windows:** double-click `index.html`, or run `start index.html`.
- **Mac:** double-click `index.html`, or run `open index.html`.

This works only when Allowed Origins is `*`.

## 9. Put your own copy on GitHub Pages
1. On GitHub, open this repository and click **Fork** (top right). You can also create a new **public** repository and upload `index.html`. GitHub Pages is free for public repositories.
2. In your copy, open **Settings > Pages**.
3. Under **Build and deployment**, set **Source** to **Deploy from a branch**, **Branch** to **main**, **Folder** to **/ (root)**, and click **Save**.
4. After about a minute, refresh. Your page is at `https://YOURNAME.github.io/REPO-NAME/`.
5. In n8n, set Allowed Origins to `https://YOURNAME.github.io` and publish again.

To change the page, edit `index.html` in your copy and commit. GitHub rebuilds it in about a minute.

## 10. Concepts

- **API:** a set of addresses a program offers so other programs can ask it for things. One address is an **endpoint**.
- **Webhook:** an endpoint that waits for requests. In n8n, the Webhook node creates one, so your workflow becomes a small API.
- **Request:** a URL, a method (GET reads, POST sends data), headers (extra details such as the data type), and a body (the data, as JSON).
- **Response:** a status code plus a body. `200` worked, `400` bad request, `401` not allowed, `404` wrong address or workflow not published, `500` the server failed.
- **JSON:** labelled data, for example `{"message": "Plan a trip..."}`.
- **CORS:** a browser rule. A page may only call another site if that site names the page's address as allowed.

**The request this page sends:**

```
POST https://YOUR-NAME.app.n8n.cloud/webhook/travel-planner
Content-Type: application/json

{ "message": "Plan a trip from SFO to Lisbon on Oct 12, 2026. ..." }
```

**The answer n8n returns when the last node finishes:**

```
200 OK
{ "output": "ITINERARY_UNDER_REVIEW: ... OVERALL_VERDICT: APPROVED" }
```

The page sends the request with `fetch` and shows the answer as plain text. It never inserts the answer as HTML.

## 11. Safety

**Is it safe that this repository is public?**
Yes. It holds no keys, no passwords and no webhook URL. A stranger who opens the page sees an empty URL box and nothing to call.

**Where is the real risk?**
In your **webhook URL**. Anyone who knows it can call it, and each call runs your workflow and uses the model API key behind it. That includes someone using PowerShell or `curl`: Allowed Origins only controls which *web pages* may call it from a browser, so it does not stop direct calls.

**How to reduce the risk**
1. **Keep the URL private.** Keep it out of screenshots, recordings and public files.
2. **Keep the workflow published only while you use it.** Switch **Publish** off afterwards. The URL then answers 404.
3. **Set a spending limit** on the model API key in your provider account. This is the strongest protection.
4. **Use a harder-to-guess Path**, for example `travel-planner-k7x92m`, and change it if the URL was exposed.
5. **For programs that call the webhook directly,** turn on **Header Auth** on the Webhook node, with a secret only those programs know. Do not use a secret in this page, because the page is public.

**Someone else wants to use this project. How does it work?**
They need their own n8n workflow with its own webhook. Then they either use the public page and allow `https://uday-ik.github.io` in their n8n, or fork this repository and run their own copy. They cannot use your webhook, because they do not have your URL.

**Why is my Allowed Origins value a GitHub address?**
It names the one website allowed to call your webhook from a browser. It does not stop anyone from opening the same page. They would only be sending requests to *their own* webhook.

## 12. Troubleshooting

| What you see | Likely cause and fix |
|---|---|
| `404 ... webhook "POST travel-planner" is not registered` | The workflow is not published, or you used the Test URL. Publish it and use the Production URL. |
| The page says "The request did not reach n8n" | CORS (Allowed Origins is missing or has a typo), a wrong URL, or the workflow is not published. Check all three. |
| Works with `*` but not with a specific address | Check for a trailing slash, a path, `http` instead of `https`, or a missing re-publish. |
| The terminal works but the page does not | CORS. The terminal does not use CORS. |
| The page works but the answer is empty | The first node is not reading the webhook message. Check the expression in step 1, item 4. |
| `curl` on Windows does something strange | In PowerShell `curl` is an alias. Use `Invoke-RestMethod` or `curl.exe`. |
| Mac says `zsh: command not found: python` | Use `python3`. |
| `Address already in use` when starting the local server | Another program uses port 8000. Use `python3 -m http.server 8001` and allow `http://localhost:8001`. |
| It takes about half a minute | Normal for several agents running in turn. |
| Works in the n8n editor but fails when published | A credential or setting is missing. Open the failed run in **Executions**. |
| The model API says quota or billing | Add credit or check the key in your provider account. |

## 13. Test prompts for the example workflow

| Test | Request | Expected |
|---|---|---|
| Happy path | SFO to Lisbon, Oct 12, 2026, 3 nights, budget $900, direct flight, wheelchair assistance, accessible hotel | FL101 + HT201 = **$830, APPROVED** |
| Other date | Same, Oct 13 | FL105 + HT201 = **$890, APPROVED** |
| Budget fails | Oct 12, 4 nights, budget $800 | **$970, NEEDS REVISION** |
| No flight fits | Oct 12, 3 nights, budget $300 | **NO_MATCHING_FLIGHT** |
| Wrong city | SFO to Paris | "No data available for this destination..." |

The page has a button for each of these.
