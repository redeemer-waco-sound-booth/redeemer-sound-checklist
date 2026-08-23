# Redeemer Sound Booth Checklist

A web app (PWA) that allows sound booth operators to photograph the completed Sunday Sound Booth Checklist and automatically log all results to a Google Sheet. No typing required — just open the app, point the camera at the checklist, and tap the button.

---

## What It Does

- Opens the phone camera
- Operator photographs the completed paper checklist
- AI reads the form and extracts: date, operator name, notes, and all 15 checkbox values
- Saves one row to a Google Sheet automatically
- Shows a confirmation screen with everything that was recorded

---

## How Operators Use It

1. Complete the paper checklist as normal
2. Open Chrome on your phone and go to the app URL (or tap the home screen icon if installed)
3. Point the camera at the checklist so the whole form is visible within the gold guide box
4. Tap the white circle button
5. Wait a few seconds — the app reads the form and saves it
6. Review the result screen to confirm everything looks right
7. Tap "Scan Another" if needed, or just close the app

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
    │  returns JSON
    ▼
Operator's Phone
    │
    │  writes one row
    ▼
Google Sheet
```

All services used are free or very low cost:
- GitHub Pages: free
- Cloudflare Workers: free (100,000 requests/day)
- Anthropic API: ~$0.01 per scan, pay as you go
- Google Sheets: free

---

## Accounts Required

All accounts are under **soundbooth@redeemerwaco.org**

| Service | Purpose | URL |
|---|---|---|
| GitHub | Hosts the app | github.com |
| Cloudflare | Secure proxy for AI API | cloudflare.com |
| Anthropic | AI that reads the checklist | console.anthropic.com |
| Google Cloud | Sheets API access | console.cloud.google.com |
| Google Sheets | Stores the data | sheets.google.com |

---

## Full Setup Instructions

Follow these steps in order. Each section builds on the previous one.

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
5. After 60 seconds, refresh — your app URL will appear, something like:
   `https://soundbooth-redeemerwaco.github.io/redeemer-sound-checklist`
6. **Save this URL** — this is what operators will use

---

### PART 2 — Anthropic API Setup

Anthropic provides the AI (Claude) that reads the checklist photos.

#### 2A — Create an Account
1. Go to **console.anthropic.com** and sign up with `soundbooth@redeemerwaco.org`
2. Verify the email address

#### 2B — Add Credit
1. Click **Plans & Billing** in the left sidebar
2. Click **Add credit** and add $10
3. This will cover approximately 1,000 scans — top up as needed

#### 2C — Create an API Key
1. Click **API Keys** in the left sidebar
2. Click **Create Key**, name it `redeemer-checklist`
3. The key appears on screen — **do not navigate away yet**
4. Click the copy icon to copy the key
5. **Immediately** paste it into a secure note or password manager
6. The key starts with `sk-ant-api03-` and is about 108 characters long
7. It will NEVER be shown again after you leave this page

> **Important:** Do not paste this key into any file that goes on GitHub. It must only go into Cloudflare (next step).

---

### PART 3 — Cloudflare Setup

Cloudflare acts as a secure middleman between the app and the AI. The API key is stored here, never in the app files on GitHub.

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
10. **Copy the Worker URL** — it looks like `https://redeemer-checklist-proxy.soundbooth-redeemerwaco.workers.dev`

#### 3C — Add the API Key as a Secret
1. Click the **Settings** tab on your Worker page
2. Click **Variables and Secrets** in the left sidebar
3. Under Secrets, click **Add**
4. Variable name: `ANTHROPIC_API_KEY`
5. Value: paste your Anthropic API key from Part 2C
6. Click **Deploy**

---

### PART 4 — Google Sheets Setup

#### 4A — Create a Google Cloud Project
1. Go to **console.cloud.google.com** and sign in with `soundbooth@redeemerwaco.org`
2. At the top, click the project dropdown → **New Project**
3. Name it `redeemer-checklist` and click **Create**
4. Make sure the new project is selected in the dropdown

#### 4B — Enable the Google Sheets API
1. In the left sidebar, click **APIs & Services** → **Library**
2. Search for `Google Sheets API`
3. Click it, then click **Enable**

#### 4C — Create a Service Account
A service account lets the app write to the sheet without anyone needing to log in.

1. In the left sidebar, click **APIs & Services** → **Credentials**
2. Scroll to the bottom — you will see a **Service Accounts** section
3. Click **+ Create Credentials** → **Service Account** at the top of the page
4. Name it `redeemer-checklist-writer`, click **Create and Continue**
5. Skip the optional steps — click **Done**
6. Back on the Credentials page, scroll to **Service Accounts** at the bottom
7. Click the service account email address
8. Click the **Keys** tab
7. Click **Add Key** → **Create new key** → **JSON** → **Create**
8. A JSON file downloads to your computer — **keep this file safe**
9. Open the file in a text editor and note two values:
   - `client_email` — looks like `redeemer-checklist-writer@redeemer-checklist.iam.gserviceaccount.com`
   - `private_key` — a long block starting with `-----BEGIN PRIVATE KEY-----`

#### 4D — Create the Google Sheet
1. Go to **sheets.google.com** and create a new blank spreadsheet
2. Name it `Sound Booth Checklist Log`
3. In Row 1, paste these headers (one per cell, A through R):

```
Date | Operator | Notes | Power supply on top of desk | Computer power on | Check mains, auxiliary speakers and subwoofer are audible | Start Spotify and make sure you can hear it | Test wireless mics | In OBS Verify "Audio Input Capture" Meter in OBS is moving and camera visible | Verify NDI Stream on TV in Lobby | Check DB Levels - should average around 80 db | At 9:45 AM, start the stream and recording | Check that livestream is available on Youtube | At 10 AM, switch the scene to the one showing cameras | When the service ends, click "Stop Streaming" and "Stop Recording" | Remove batteries from microphones and put in charger | After service, trim the video and publish | Shut down the computer and sound board
```

4. Look at the URL — copy the Sheet ID from it:
   `https://docs.google.com/spreadsheets/d/`**`THIS_PART_IS_THE_SHEET_ID`**`/edit`

#### 4E — Share the Sheet with the Service Account
1. Click **Share** (top right of the sheet)
2. Paste the `client_email` from the JSON file into the "Add people" box
3. Set role to **Editor**
4. Uncheck "Notify people"
5. Click **Share**

---

### PART 5 — Configure and Upload the App

#### 5A — Edit index.html
Open `index.html` in a text editor and find this section near the top of the script:

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
- `YOUR_CLOUDFLARE_WORKER_URL` → the Worker URL from Part 3B
- `YOUR_GOOGLE_SHEET_ID` → the Sheet ID from Part 4D
- `YOUR_SERVICE_ACCOUNT_EMAIL` → the `client_email` from the JSON file
- `YOUR_PRIVATE_KEY` → the `private_key` from the JSON file (include the full `-----BEGIN...-----END-----` block)

> **Note:** The Anthropic API key does NOT go here. It lives only in Cloudflare.

#### 5B — Upload Files to GitHub
Upload all four files to the GitHub repository:
- `index.html` (with your credentials filled in)
- `manifest.json`
- `README.md`
- `icon-192.png` and `icon-512.png` (generate free at favicon.io — download, rename files)

To upload: go to your GitHub repo → **Add file** → **Upload files** → drag files in → **Commit changes**.

GitHub Pages will update within 60 seconds of each upload.

---

### PART 6 — Test It

1. On a phone, open **Chrome** and go to the app URL
2. Allow camera permission when asked
3. Point the camera at a completed checklist and tap the button
4. After a few seconds you should see the result screen with all items listed
5. Check the Google Sheet — a new row should appear

**Confirm you're on the latest version** by checking the build timestamp in the top-right corner of the header. It should match the most recently deployed version.

---

### PART 7 — Add the QR Code to the Checklist

Once the app is live and tested:
1. Go to **qr-code-generator.com** or any free QR code generator
2. Enter the GitHub Pages URL
3. Download the QR code image
4. Add it to the bottom of the checklist document where it says "Scan the QR code to upload this form"
5. Reprint the checklist

---

## Maintenance

### If the AI stops working (API error)
The Anthropic API key may need to be replaced or the credit balance topped up.
1. Go to **console.anthropic.com** → **Plans & Billing** to check the balance
2. If the key needs replacing: create a new key, go to Cloudflare → Worker → Settings → Variables and Secrets → edit `ANTHROPIC_API_KEY`

### If the sheet fills up
The Google Sheet can hold thousands of rows. If it ever gets unwieldy, create a new sheet, share it with the service account, update `SHEET_ID` in `index.html`, and re-upload to GitHub.

### If the checklist changes
If items are added, removed, or reworded:
1. Update the `CHECKLIST_ITEMS` array in `index.html` to match the new list exactly
2. Update the column headers in the Google Sheet to match
3. Re-upload `index.html` to GitHub

### If you need to transfer ownership
All accounts are under `soundbooth@redeemerwaco.org`. Transfer access by sharing that account's credentials with the new administrator, or create new accounts and repeat the setup steps above.

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| Camera doesn't appear | Not using Chrome, or permission denied | Use Chrome; tap "Try Again" on error screen |
| "API error" on scan | Anthropic key missing or expired | Check Cloudflare secret; check Anthropic billing balance |
| Result screen shows wrong data | Photo was blurry or poorly framed | Try again with better lighting and steady hands |
| Nothing appears in the sheet | Sheet not shared with service account | Share the sheet with the service account email as Editor |
| App shows old version | Browser cache | Clear Chrome cache or re-enter the URL manually |
| "Google auth failed" | Service account credentials wrong | Re-check client_email and private_key in index.html |

---

## File Reference

| File | Purpose |
|---|---|
| `index.html` | The entire app — all code, styles, and logic |
| `manifest.json` | Makes the app installable to phone home screen |
| `README.md` | This document |
| `icon-192.png` | App icon (small) |
| `icon-512.png` | App icon (large) |
