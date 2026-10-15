# Accessible Travel Planner (web page for an n8n workflow)

A single web page that sends a trip request to an n8n workflow through its **webhook** and shows the checked plan that comes back. The page is only the front door. The team of AI agents runs in n8n.

Live page: https://uday-ik.github.io/travel-planner-demo/

## How to use it
1. Build the multi-agent travel planning workflow in n8n (Week 3 demo guide) and add a **Webhook** node: method POST, path `travel-planner`, respond "When Last Node Finishes", response data "First Entry JSON".
2. In the Webhook node, open Options and set **Allowed Origins (CORS)** to `https://uday-ik.github.io` (the address of this page, with no path). Use `*` only for a quick test.
3. **Publish** the workflow. The webhook only exists while it is published.
4. Open the page, paste the **Production** webhook URL, and click **Plan my trip**.

## What is in this repo
- `index.html`: the whole page (HTML, CSS and JavaScript in one file, no build step). It sends `POST {"message": "..."}` and shows the `output` field of the JSON answer.

## Safety
- The page holds no keys or secrets. The webhook URL is typed by the person using the page and is kept only in their own browser.
- Anyone who has a webhook URL can call it and spend the API key behind that workflow. Set a spending limit on the key and switch **Publish** off when you are not using it.
- CORS only protects against other websites in a browser. For apps that call the webhook directly, use Header Auth and never put the secret in this page.
