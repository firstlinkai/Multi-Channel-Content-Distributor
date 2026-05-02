# 🏝️ Multi-Channel Content Distributor

> **Post Once. Reach Everywhere.**
> An intelligent n8n automation that transforms a single resort photo or video into three fully-crafted, platform-optimized social media posts — published or staged for review across Facebook, Instagram, and TikTok automatically.

***

## 📋 Table of Contents

- [Overview](#overview)
- [How It Works — At a Glance](#how-it-works--at-a-glance)
- [Workflow Architecture](#workflow-architecture)
- [Node-by-Node Breakdown](#node-by-node-breakdown)
- [AI Caption Strategy](#ai-caption-strategy)
- [Platform Behavior & Important Notes](#platform-behavior--important-notes)
- [Tech Stack](#tech-stack)
- [Prerequisites & Setup](#prerequisites--setup)
- [Configuration Reference](#configuration-reference)
- [Outputs & Logging](#outputs--logging)
- [File Structure](#file-structure)
- [Known Limitations](#known-limitations)

***

## Overview

**Project Code:** `RRE-AUTO-03`
**Workflow Name:** Multi-Channel Content Distributor
**Status:** Production-Ready Blueprint
**Built For:** Rosario Resort & Estates — Cavite, Philippines

Resort marketing teams lose hours every week manually resizing, rewriting, and rescheduling the same content for different platforms. This system eliminates that bottleneck entirely.

Drop a photo or video into a designated Google Drive folder → the automation downloads the file, sends it through GPT-4o Vision for AI-powered scene analysis, generates three platform-specific captions (Facebook, Instagram, TikTok), routes the media to each platform's API, and logs everything in Google Sheets while sending a Slack notification — all without human intervention.

**Estimated time saved:** ~70% reduction in social media marketing labor per post.

***

## How It Works — At a Glance

```
📁 Google Drive Upload
        │
        ▼
🔍 GPT-4o Vision Analysis
        │  "A golden sunset behind a bamboo cabana..."
        ▼
✍️  GPT-4o Caption Generator
        │
        ├──▶ 📘 Facebook Caption  (150-200 words, storytelling, booking CTA)
        ├──▶ 📸 Instagram Caption (60-80 words, aesthetic, emojis, hashtags)
        └──▶ 🎵 TikTok Caption   (40-60 words, Gen-Z hook, trending keywords)
        │
        ▼
🔀 Router: Image or Video?
        │
        ├── IMAGE PATH ──▶ FB Photo Draft + IG Container → Wait 30s → IG Publish
        └── VIDEO PATH ──▶ FB Video Draft + TikTok Draft
        │
        ▼
📊 Build Summary Report
        │
        ├──▶ 💬 Slack Notification
        └──▶ 📋 Google Sheets Log
```

***

## Workflow Architecture

Below is the visual schema of the complete n8n workflow:



The workflow is split into **four logical phases**:

| Phase | Nodes | Purpose |
|-------|-------|---------|
| **Ingestion** | Watch Folder → Download → Prepare Metadata | Detect and fetch new assets |
| **AI Processing** | Vision Analysis → Extract Description → Generate Captions → Parse Captions | Understand and write platform content |
| **Distribution** | Router → FB / IG / TikTok upload nodes | Deliver content to each platform |
| **Reporting** | Build Summary → Slack → Google Sheets | Log results and notify the team |

***

## Node-by-Node Breakdown

### Phase 1 — Ingestion

#### 1. `Google Drive - Watch Folder`
- **Type:** `n8n-nodes-base.googleDriveTrigger`
- **Trigger:** Polls every minute for newly created files in the configured folder
- **Event:** `fileCreated`
- **Config Required:** Set `YOUR_GOOGLE_DRIVE_FOLDER_ID` to your target Drive folder's ID
- **Credential:** Google Drive OAuth2

#### 2. `Google Drive - Download File`
- **Type:** `n8n-nodes-base.googleDrive`
- **Operation:** Downloads the detected file as binary data
- **Input:** `$json.id` from the trigger node
- **Output:** Binary file data passed forward in the pipeline

#### 3. `Prepare Asset Metadata`
- **Type:** Code node (JavaScript)
- **Purpose:** Extracts and normalizes file metadata before AI processing
- **Outputs the following fields:**

```json
{
  "fileId": "string",
  "fileName": "string",
  "mimeType": "image/jpeg | video/mp4 | ...",
  "isVideo": true | false,
  "isImage": true | false,
  "webViewLink": "string",
  "base64Data": "string",
  "assetType": "image | video"
}
```

***

### Phase 2 — AI Processing

#### 4. `GPT-4o Vision - Analyze Asset`
- **Type:** HTTP Request → `POST https://api.openai.com/v1/chat/completions`
- **Model:** `gpt-4o`
- **Max Tokens:** 500
- **Input:** Base64-encoded image with `detail: "high"` flag
- **System Prompt Role:** Resort marketing expert analyzing visual assets
- **Output:** A 2-3 sentence vivid scene description (scene, mood, lighting, colors)

> **Example Output:** *"A golden hour sunset casts warm amber light across a bamboo cabana perched above still waters. The scene radiates serenity, with lush tropical foliage framing the structure. Rich orange and violet sky tones reflect on the calm surface, creating a deeply luxurious, escapist atmosphere."*

#### 5. `Extract Vision Description`
- **Type:** Code node (JavaScript)
- **Purpose:** Parses the GPT-4o API response and merges `visualDescription` back into the asset metadata object for clean downstream use

#### 6. `GPT-4o - Generate Platform Captions`
- **Type:** HTTP Request → `POST https://api.openai.com/v1/chat/completions`
- **Model:** `gpt-4o`
- **Max Tokens:** 1500
- **Response Format:** `json_object` (enforced structured output)
- **System Persona:** Expert social media manager for Rosario Resort & Estates

The node sends the visual description and instructs GPT-4o to generate three distinct captions in a single API call. See [AI Caption Strategy](#ai-caption-strategy) for full prompt details.

#### 7. `Parse & Format Captions`
- **Type:** Code node (JavaScript)
- **Purpose:** Parses the JSON response from GPT-4o, extracts `facebook`, `instagram`, and `tiktok` caption objects, and merges hashtags into final caption strings ready for API submission

***

### Phase 3 — Distribution

#### 8. `Route: Image or Video?`
- **Type:** Switch node
- **Condition:** Checks `assetType` field
- **Image Route (Output 0):** → Facebook Photo Draft + Instagram Container
- **Video Route (Output 1):** → Facebook Video Draft + TikTok Draft

***

#### IMAGE ROUTE

##### 9a. `Facebook - Upload Image Draft`
- **Endpoint:** `POST https://graph.facebook.com/{PAGE_ID}/photos`
- **Mode:** `published: false` — posts land as **unpublished drafts** for review
- **Payload:** Binary image + Facebook caption string

##### 9b. `Instagram - Create Media Container`
- **Endpoint:** `POST https://graph.facebook.com/{IG_USER_ID}/media`
- **Purpose:** Creates the media container (required first step of IG's 2-step publish API)
- **Payload:** Image URL + caption string

##### 10. `Store IG Container ID`
- **Type:** Code node
- **Purpose:** Captures the `id` from the Instagram container creation response for use in the publish step

##### 11. `Wait 30s - IG Processing`
- **Type:** Wait node — 30-second pause
- **Reason:** Instagram's Graph API requires processing time between container creation and publishing. This is a **mandatory delay** enforced by Meta.

##### 12. `Instagram - Publish Post`
- **Endpoint:** `POST https://graph.facebook.com/{IG_USER_ID}/media_publish`
- **Payload:** `creation_id` from the stored container ID

***

#### VIDEO ROUTE

##### 9c. `Facebook - Upload Video Draft`
- **Endpoint:** `POST https://graph.facebook.com/{PAGE_ID}/videos`
- **Mode:** `published: false` — videos land as **unpublished drafts** for review
- **Payload:** Binary video file + Facebook caption

##### 9d. `TikTok - Upload Draft`
- **Endpoint:** `POST https://open.tiktokapis.com/...`
- **Mode:** `SELF_ONLY` privacy level (draft mode — safe for review before going public)
- **Note:** Change `privacy_level` to `PUBLIC_TO_EVERYONE` when ready to publish live

***

### Phase 4 — Reporting

#### 13. `Build Summary Report`
- **Type:** Code node (JavaScript)
- **Purpose:** Compiles a structured JSON summary of the entire run including asset name, captions generated, platforms reached, and timestamp

#### 14. `Slack - Send Success Notification`
- **Endpoint:** `POST https://hooks.slack.com/...` (Slack Incoming Webhook)
- **Purpose:** Sends a rich notification to your designated Slack channel confirming successful distribution with a summary of what was posted

#### 15. `Google Sheets - Log Content`
- **Operation:** `appendOrUpdate`
- **Sheet Name:** `Content Log` *(must be created manually — see setup)*
- **Purpose:** Appends one row per run with all metadata fields, enabling a running content audit trail

***

## AI Caption Strategy

All three captions are generated in a **single GPT-4o API call** using structured JSON output mode. Each caption is purpose-built for its platform's algorithm and audience behavior.

### Facebook Caption
```
Length:   150–200 words
Tone:     Warm, storytelling, informative
Includes: Scene description, guest experience narrative,
          amenity mentions, location tag (Cavite, Philippines),
          booking CTA with [WEBSITE_URL] placeholder, 2–3 hashtags
Goal:     Drive link clicks and direct booking inquiries
```

### Instagram Caption
```
Length:   60–80 words
Tone:     Aesthetic, dreamy, aspirational
Includes: Powerful opening one-liner, 5–7 visual emojis woven naturally,
          engagement question at the end
Hashtags: #RosarioResort #ResortLife #TropicalParadise #LuxuryTravel
          #PhilippineBeauty #IslandVibes #VacationGoals #HotelLife
          #TravelPH #LuxuryResort
Goal:     Maximize saves, shares, and comment engagement
```

### TikTok Caption
```
Length:   40–60 words
Tone:     High-energy, Gen Z, raw and authentic
Includes: CAPS hook (bold statement or provocative question),
          trending keywords (POV, no cap, fr fr, telling me),
          challenge or duet invite at the end
Hashtags: #RosarioResort #ResortTok #PhilippineTravel #TravelTok
          #LuxuryResortCheck #FYP #PinoyTravel
Goal:     Trigger algorithmic push via FYP discovery
```

***

## Platform Behavior & Important Notes

> ⚠️ Read carefully before going live.

| Platform | Behavior | Action Required |
|----------|----------|-----------------|
| **Facebook (Images)** | Posts as **unpublished draft** (`published: false`) | Review in Meta Business Suite → publish manually |
| **Facebook (Videos)** | Posts as **unpublished draft** | Review in Meta Business Suite → publish manually |
| **Instagram** | Fully published after the 30s wait | No action needed — posts go live automatically |
| **TikTok** | Lands in **SELF_ONLY** (creator draft mode) | Change `privacy_level` to `PUBLIC_TO_EVERYONE` in the TikTok node to go live |
| **Google Sheets** | Auto-logs every run | Create a sheet named `Content Log` with headers matching node field names |

***

## Tech Stack

| Layer | Tool |
|-------|------|
| **Workflow Orchestrator** | [n8n](https://n8n.io) (self-hosted or cloud) |
| **Asset Storage & Trigger** | Google Drive |
| **AI Vision & Caption Engine** | OpenAI GPT-4o (`gpt-4o`) |
| **Facebook Distribution** | Meta Graph API (Pages & Photos endpoints) |
| **Instagram Distribution** | Meta Graph API (Instagram Business API — 2-step publish) |
| **TikTok Distribution** | TikTok Business API (`open.tiktokapis.com`) |
| **Team Notifications** | Slack Incoming Webhooks |
| **Content Audit Log** | Google Sheets API |

***

## Prerequisites & Setup

### Accounts & Access Required

- [ ] n8n instance (self-hosted on AWS/VPS or n8n Cloud)
- [ ] Google account with Drive API enabled
- [ ] OpenAI account with GPT-4o API access
- [ ] Facebook Developer App with `pages_manage_posts` and `pages_read_engagement` permissions
- [ ] Instagram Business Account connected to your Facebook Page
- [ ] TikTok Business API developer account
- [ ] Slack workspace with an Incoming Webhook URL configured
- [ ] Google Sheets document with a sheet named `Content Log`

### Step-by-Step Installation

**1. Import the Blueprint**

In your n8n instance:
```
Settings → Import Workflow → Upload File
```
Select `RRE-AUTO-03-_-Multi-Channel-Content-Distributor.json`

**2. Configure Credentials**

Set up the following credentials in n8n (`Settings → Credentials`):

| Credential Name | Type | Notes |
|-----------------|------|-------|
| `Google Drive account` | Google Drive OAuth2 | Needs Drive read access |
| `OpenAI API` | HTTP Header Auth | Header: `Authorization`, Value: `Bearer YOUR_KEY` |
| `Facebook Page Token` | HTTP Header Auth | Page access token from Meta Developer Console |
| `TikTok API` | HTTP Header Auth | From TikTok Business API dashboard |
| `Slack Webhook` | HTTP Request | Incoming Webhook URL from Slack App settings |
| `Google Sheets` | Google Sheets OAuth2 | Needs Sheets read/write access |

**3. Update Configuration Values**

Open the workflow and update the following placeholders:

```
Google Drive - Watch Folder node:
  → folderToWatch.value = "YOUR_GOOGLE_DRIVE_FOLDER_ID"

Facebook - Upload Image/Video Draft nodes:
  → URL: Replace {PAGE_ID} with your Facebook Page ID

Instagram nodes:
  → URL: Replace {IG_USER_ID} with your Instagram Business Account ID

Google Sheets - Log Content node:
  → Spreadsheet ID and Sheet Name = "Content Log"

GPT-4o Caption Generator:
  → Replace [WEBSITE_URL] in the prompt with your actual resort booking URL
```

**4. Create the Google Sheets Log**

Create a new Google Sheet named `Content Log` with the following column headers:
```
Timestamp | File Name | Asset Type | File ID | Visual Description | FB Caption | IG Caption | TikTok Caption | Platforms Reached | Status
```

**5. Activate the Workflow**

Toggle the workflow to **Active** in n8n. It will begin polling Google Drive every minute for new files.

***

## Configuration Reference

| Parameter | Location | Default | Description |
|-----------|----------|---------|-------------|
| `folderToWatch` | Watch Folder node | `YOUR_GOOGLE_DRIVE_FOLDER_ID` | Google Drive folder ID to monitor |
| `pollTimes.mode` | Watch Folder node | `everyMinute` | How often to check for new files |
| `model` (Vision) | GPT-4o Vision node | `gpt-4o` | OpenAI model for image analysis |
| `max_tokens` (Vision) | GPT-4o Vision node | `500` | Token limit for scene description |
| `model` (Captions) | GPT-4o Caption node | `gpt-4o` | OpenAI model for caption generation |
| `max_tokens` (Captions) | GPT-4o Caption node | `1500` | Token limit for 3-platform captions |
| `published` | FB Draft nodes | `false` | Set `true` to auto-publish on Facebook |
| `privacy_level` | TikTok node | `SELF_ONLY` | Set `PUBLIC_TO_EVERYONE` for live TikTok posts |
| `waitAmount` | Wait node | `30 seconds` | Instagram API processing buffer |

***

## Outputs & Logging

Every time a file is uploaded to the watched Drive folder, the following outputs are produced:

### 1. Facebook
- ✅ Image → Unpublished photo draft (visible in Meta Business Suite → Content Calendar)
- ✅ Video → Unpublished video draft (visible in Meta Business Suite → Video Library)

### 2. Instagram
- ✅ Image → Published post (live immediately after 30s processing)
- ⚠️ Video → Requires Instagram Reels endpoint modification for video support

### 3. TikTok
- ✅ Video → Draft post (visible in TikTok Creator Studio → Drafts)
- ⚠️ Image → TikTok is video-first; image posts may require slideshow format handling

### 4. Slack Notification
A message is sent to your configured Slack channel confirming:
- File name processed
- Asset type (image/video)
- Platforms reached
- AI-generated caption previews

### 5. Google Sheets Content Log
One row appended per run:

```
| 2026-05-02 13:00 | sunset-cabana.jpg | image | 1xAbc... | A golden sunset... | [FB caption] | [IG caption] | [TikTok caption] | FB, IG, TikTok | Success |
```

***

## File Structure

```
RRE-AUTO-03/
├── README.md                                          ← This file
├── RRE-AUTO-03-_-Multi-Channel-Content-Distributor.json   ← n8n workflow blueprint
└── RRE-AUTO-03-_-Multi-Channel-Content-Distributor-Schema.jpg  ← Workflow diagram
```

***
<img width="1370" height="697" alt="RRE-AUTO-03 _ Multi-Channel Content Distributor Schema" src="https://github.com/user-attachments/assets/b1033255-7435-4f13-9017-48ac1b094838" />


## Known Limitations

| Limitation | Detail | Workaround |
|------------|--------|------------|
| **Instagram Videos** | The current IG path handles images only (photo container endpoint) | Switch to Reels endpoint (`/reels`) for video uploads |
| **TikTok Images** | TikTok's API is video-first; static images aren't directly supported | Use TikTok's photo slideshow API or skip TikTok for image assets |
| **GPT-4o Vision on Videos** | GPT-4o Vision analyzes the file as-is; for videos it processes only the first frame/thumbnail | Pre-extract a keyframe before the Vision node for better accuracy |
| **Drive Polling Frequency** | Default polling is every minute — may pick up partially uploaded large video files | Add a file size or naming convention check in the Prepare Metadata node |
| **Facebook Auto-Publish** | Facebook posts land as drafts by default to prevent accidental publishing | Set `published: true` in both Facebook nodes when confident in the AI output |
| **TikTok Privacy** | TikTok posts are `SELF_ONLY` by default | Change `privacy_level` to `PUBLIC_TO_EVERYONE` after reviewing drafts |
| **Rate Limits** | High-volume uploads may hit OpenAI or Meta API rate limits | Add retry logic or throttle the Drive folder upload frequency |

***

## 📄 License

This workflow was built for **Rosario Resort & Estates** as part of the RRE Automation Series. You are free to adapt it for your own resort, hotel, or hospitality brand. Attribution appreciated but not required.
