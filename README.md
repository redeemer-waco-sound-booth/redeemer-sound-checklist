# Redeemer Sound Booth Checklist

A web app (PWA) that allows sound booth operators to photograph the completed Sunday Sound Booth Checklist, review and correct what the AI read, then save it to a Google Sheet with one tap.

---

## What It Does

1. Opens the phone camera
2. Operator photographs the completed paper checklist
3. AI reads the form and extracts: date, operator name, notes, and all 15 checkbox values
4. A review screen appears showing everything the AI read — all fields are editable
5. Operator corrects any mistakes, toggles any misread checkboxes
6. Operator taps Submit to save one row to the Google Sheet

---

## How Operators Use It

1. Complete the paper checklist as normal
2. Open Chrome on your phone and go to the app URL (or tap the home screen icon if installed)
3. Point the camera at the checklist so the whole form fits within the gold guide box
4. Tap the white circle button
5. Wait a few seconds while the AI reads the form
6. Review screen appears — check that everything looks right
7. Tap any checklist item to toggle it between ✓ and ✗ if it was misread
8. Edit the date, operator name, or notes fields if needed
9. Tap **Submit →** to save to the spreadsheet
10. Green confirmation appears — tap **Scan Another →** if needed

To install to home screen: in Chrome, tap the three-dot menu → Add to Home Screen → Add.

---

## Checklist Items Tracked

The app reads and records these 15 items as TRUE or FALSE:

1. Power supply on top of desk
2. Computer power on
3. Check mains, auxiliary speakers and subwoofer are audible
4. Start Spotify and make sure you can hear it
5. Test wireless mics
6. In OBS Verify "Audio Input Capture" Meter in OBS is moving and camera visible
7. Verify NDI Stream on TV in Lobby
8. Check DB Levels - should average around 80 db
9. At 9:45 AM, start the stream and recording
10. Check that livestream is available on Youtube
11. At 10 AM, switch the scene to the one showing cameras
12. When the service ends, click "Stop Streaming" and "Stop Recording"
13. Remove batteries from microphones and put in charger
14. After service, trim the video and publish
15. Shut down the computer and sound board

Plus: Date, Operator Name, and Problems/Notes as free text.

---

## Architecture Overview

```
Operator's Phone (Chrome PWA)
    │
    │  sends photo to proxy server
    ▼
Cloudflare Worker  ← holds Anthropic API key securely
    │
    │  forwards to Claude AI
    ▼
Anthropic Claude API  ← reads the form and returns structured data
    │
    │  returns JSON to app
    ▼
Operator's Phone (review screen)
    │
    │  operator edits if needed, taps Submit
    │  writes one row via Google Sheets API
    ▼
Google Sheet
```

All services used are free or very low cost:
- GitHub Pages: free
- Cloudflare Workers: free (100,000 requests/day)
- Anthropic API: ~$0.01 per scan, pay as you go
- Google Sheets: free

---

## Accounts

All accounts are under **soundbooth@redeemerwaco.org**

| Service | Purpose | URL |
|---|---|---|
| GitHub | Hosts the app | github.com |
| Cloudflare | Secure proxy for AI API | cloudflare.com |
| Anthropic | AI that reads the checklist | console.anthropic.com |
| Google Cloud | Sheets API access | console.cloud.google.com |
| Google Sheets | Stores the data | sheets.google.com |

---

## Credentials Reference

Keep this information in a secure location. These are needed to maintain or rebuild the app.

| Item | Value |
|---|---|
| Cloudflare Worker URL | https://steep-wind-3233.soundbooth.workers.dev/ |
| Google Sheet ID | 19S6KzqLBFq3BNU3TSVla_tWnZUpGvefXbCeOyx74iVQ |
| Service Account Email | redeemer-checklist-writer@redeemer-checklist.iam.gserviceaccount.com |
| Service Account Private Key | In the downloaded JSON file — see Google Cloud setup |
| Anthropic API Key | Stored only in Cloudflare Worker secret — never in this repo |

---

## Full Setup Instructions

Follow these steps in order if rebuilding from scratch. Each section builds on the previous one.

---

### PART 1 — GitHub Setup

GitHub hosts the app files and makes them available as a website for free.

#### 1A — Create a GitHub Account
1. Go to **github.com** and sign up using `soundbooth@redeemerwaco.org`
2. Verify the email address when prompted

#### 1B — Create the Repository
1. Click the **+** icon (top right) → **New repository**
2. Name it exactly: `redeemer-sound-checklist`
3. Set visibility to **Public**
4. Check **Add a README file**
5. Click **Create repository**

#### 1C — Enable GitHub Pages
1. In the repository, click **Settings** (top tab)
2. In the left sidebar, click **Pages**
3. Under Source, select **Deploy from a branch**
4. Under Branch, select **main** and **/ (root)**, click **Save**
5. After 60 seconds, refresh — your app URL will appear:
   `https://soundbooth-redeemerwaco.github.io/redeemer-sound-checklist`
6. **Save this URL** — this is what operators will use

#### 1D — Upload App Files
Upload all five files to the repository via **Add file → Upload files**:
- `index.html`
- `manifest.json`
- `README.md`
- `icon-192.png`
- `icon-512.png`

Icons can be generated free at **favicon.io/favicon-generator** — download the ZIP, rename `android-chrome-192x192.png` → `icon-192.png` and `android-chrome-512x512.png` → `icon-512.png`.

---

### PART 2 — Anthropic API Setup

Anthropic provides the AI (Claude) that reads the checklist photos.

#### 2A — Create an Account
1. Go to **console.anthropic.com** and sign up with `soundbooth@redeemerwaco.org`
2. Verify the email address

#### 2B — Add Credit
1. Click **Plans & Billing** in the left sidebar
2. Click **Add credit** and add $10
3. This covers approximately 1,000 scans — top up as needed

#### 2C — Create an API Key
1. Click **API Keys** in the left sidebar
2. Click **Create Key**, name it `redeemer-checklist`
3. The key appears on screen — **do not navigate away yet**
4. Click the copy icon to copy the key
5. **Immediately** open Cloudflare (Part 3) and paste it there as a secret
6. The key starts with `sk-ant-api03-` and is about 108 characters long
7. It will NEVER be shown again after you leave this page

> **Critical:** Do not paste this key into index.html or any file on GitHub. It must only go into Cloudflare as a secret, or Anthropic will automatically revoke it within minutes.

---

### PART 3 — Cloudflare Setup

Cloudflare acts as a secure middleman. The Anthropic API key is stored here — never in the app files on GitHub.

#### 3A — Create an Account
1. Go to **cloudflare.com** and sign up with `soundbooth@redeemerwaco.org`

#### 3B — Create a Worker
1. In the left sidebar, click **Compute** → **Workers & Pages**
2. Click **Create application**
3. Click **Start with Hello World!**
4. Name it `redeemer-checklist-proxy`
5. Click **Deploy**
6. Click **Edit Code**
7. Select all the default code and delete it
8. Paste in exactly this code:

```javascript
export default {
  async fetch(request, env) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        }
      });
    }

    const body = await request.json();

    const response = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': env.ANTHROPIC_API_KEY,
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify(body)
    });

    const data = await response.json();

    return new Response(JSON.stringify(data), {
      status: response.status,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
      }
    });
  }
};
```

9. Click **Deploy**
10. **Copy the Worker URL** — shown at the top of the page, looks like:
    `https://steep-wind-3233.soundbooth.workers.dev/`
    Include the trailing slash.

#### 3C — Add the API Key as a Secret
1. Click the **Settings** tab on your Worker page
2. Click **Variables and Secrets** in the left sidebar
3. Under Secrets, click **Add**
4. Set Type to **Secret** (not Text)
5. Variable name: `ANTHROPIC_API_KEY`
6. Value: paste your Anthropic API key from Part 2C
7. Click **Deploy**

---

### PART 4 — Google Sheets Setup

#### 4A — Create a Google Cloud Project
1. Go to **console.cloud.google.com** and sign in with `soundbooth@redeemerwaco.org`
2. At the top, click the project dropdown → **New Project**
3. Name it `redeemer-checklist` and click **Create**
4. Make sure the new project is selected in the dropdown at the top

#### 4B — Enable the Google Sheets API
1. In the left sidebar, click **APIs & Services** → **Library**
2. Search for `Google Sheets API`
3. Click it, then click **Enable**

#### 4C — Create a Service Account
1. In the left sidebar, click **APIs & Services** → **Credentials**
2. Click **+ Create Credentials** → **Service Account**
3. Name it `redeemer-checklist-writer`, click **Create and Continue**
4. Skip the optional steps — click **Done**
5. Back on the Credentials page, scroll to **Service Accounts** at the bottom
6. Click the service account email address
7. Click the **Keys** tab
8. Click **Add Key** → **Create new key** → **JSON** → **Create**
9. A JSON file downloads to your computer — **keep this file safe, it cannot be regenerated**
10. Open the file in a text editor and note two values:
    - `client_email` — looks like `redeemer-checklist-writer@redeemer-checklist.iam.gserviceaccount.com`
    - `private_key` — the entire block from `-----BEGIN PRIVATE KEY-----` to `-----END PRIVATE KEY-----\n`

#### 4D — Create the Google Sheet
1. Go to **sheets.google.com** and create a new blank spreadsheet
2. Name it `Sound Booth Checklist Log`
3. In Row 1, add these headers across columns A through R:

```
Date | Operator | Notes | Power supply on top of desk | Computer power on | Check mains, auxiliary speakers and subwoofer are audible | Start Spotify and make sure you can hear it | Test wireless mics | In OBS Verify "Audio Input Capture" Meter in OBS is moving and camera visible | Verify NDI Stream on TV in Lobby | Check DB Levels - should average around 80 db | At 9:45 AM, start the stream and recording | Check that livestream is available on Youtube | At 10 AM, switch the scene to the one showing cameras | When the service ends, click "Stop Streaming" and "Stop Recording" | Remove batteries from microphones and put in charger | After service, trim the video and publish | Shut down the computer and sound board
```

4. Get the Sheet ID from the URL:
   `https://docs.google.com/spreadsheets/d/`**`SHEET_ID_IS_HERE`**`/edit`

#### 4E — Share the Sheet with the Service Account
1. Click **Share** (top right of the sheet)
2. Paste the `client_email` from the JSON file into the "Add people" box
3. Set role to **Editor**
4. Uncheck "Notify people"
5. Click **Share**

---

### PART 5 — Configure and Upload index.html

Open `index.html` in a text editor and find the CONFIG block near the top of the script section:

```javascript
const CONFIG = {
  WORKER_URL: 'YOUR_CLOUDFLARE_WORKER_URL',
  SHEET_ID:   'YOUR_GOOGLE_SHEET_ID',
  SERVICE_ACCOUNT: {
    client_email: 'YOUR_SERVICE_ACCOUNT_EMAIL',
    private_key:  'YOUR_PRIVATE_KEY',
  }
};
```

Replace each placeholder:
- `YOUR_CLOUDFLARE_WORKER_URL` → Worker URL from Part 3B (include trailing slash)
- `YOUR_GOOGLE_SHEET_ID` → Sheet ID from Part 4D
- `YOUR_SERVICE_ACCOUNT_EMAIL` → `client_email` from the JSON file
- `YOUR_PRIVATE_KEY` → the full `private_key` value from the JSON file

> **Private key note:** Copy the entire value from the JSON file including `-----BEGIN PRIVATE KEY-----` and `-----END PRIVATE KEY-----\n` and all the `\n` characters within it. Do not copy just the key ID (a short hex string) — that is not the private key.

> **The Anthropic API key does NOT go here.** It lives only in Cloudflare.

Upload the completed `index.html` to GitHub via **Add file → Upload files**.

---

### PART 6 — Test

1. On a phone, open **Chrome** and navigate to the GitHub Pages URL
2. Allow camera permission when asked
3. Point the camera at a completed checklist and tap the shutter button
4. After a few seconds the review screen should appear
5. Verify the data looks correct, tap **Submit →**
6. Check the Google Sheet — a new row should appear

**Confirm you're on the latest version** by checking the build timestamp in the top-right corner of the header.

---

### PART 7 — Add the QR Code to the Checklist

Once the app is live and tested:
1. Go to any free QR code generator (e.g. **qr-code-generator.com**)
2. Enter the GitHub Pages URL
3. Download the QR code image
4. Add it to the checklist document where it says "Scan the QR code to upload this form"
5. Reprint the checklist

---

## Maintenance

### Topping up Anthropic credit
Go to **console.anthropic.com** → **Plans & Billing** → **Add credit**. Each scan costs roughly $0.01.

### Replacing the Anthropic API key
If the key stops working: create a new key at console.anthropic.com, then go to Cloudflare → Compute → Workers & Pages → `redeemer-checklist-proxy` → Settings → Variables and Secrets → edit `ANTHROPIC_API_KEY`. Never put the key in index.html.

### If the checklist changes
1. Update the `CHECKLIST_ITEMS` array in `index.html` to match the new list exactly
2. Update the column headers in the Google Sheet to match
3. Re-upload `index.html` to GitHub

### If the sheet gets too large
Create a new Google Sheet, share it with the service account email as Editor, update `SHEET_ID` in `index.html`, re-upload to GitHub.

### Transferring ownership
All accounts are under `soundbooth@redeemerwaco.org`. Share that account's credentials with the new administrator, or set up new accounts and follow the setup steps above.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| Camera doesn't appear | Not using Chrome, or permission denied | Use Chrome; tap "Try Again" on error screen |
| "Failed to fetch" | Worker URL wrong or Worker not deployed | Check WORKER_URL in index.html includes trailing slash; verify Worker is deployed in Cloudflare |
| "API error 405" | Worker URL missing trailing slash | Add `/` to end of WORKER_URL in index.html |
| "API error" on scan | Anthropic key missing, expired, or wrong | Check Cloudflare secret is type Secret not Text; check Anthropic billing balance |
| "Google auth failed" | Private key wrong or malformed | Re-check private_key in index.html — must be the full key block, not the key ID |
| Nothing in the sheet | Sheet not shared with service account | Share sheet with service account email as Editor |
| Review screen shows wrong data | Photo blurry or poorly framed | Rescan with better lighting and steady hands |
| App shows old version | Browser cache | Clear Chrome cache or re-enter the URL manually |
| Anthropic key got revoked | Key was committed to public GitHub repo | Never put the key in index.html; keep it only in Cloudflare |

---

## File Reference

| File | Purpose |
|---|---|
| `index.html` | The entire app — all code, styles, and logic |
| `manifest.json` | Makes the app installable to phone home screen |
| `README.md` | This document |
| `icon-192.png` | App icon (small) |
| `icon-512.png` | App icon (large) |
