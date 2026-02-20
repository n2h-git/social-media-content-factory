# AI Social Media Content Factory

An AI-powered n8n workflow that generates, reviews, and publishes social media content across multiple platforms from a single topic input. Supports both scheduled (cron + RSS) and manual (form) content creation with human-in-the-loop approval via email.

## Architecture

```
┌─────────────┐     ┌──────────────┐
│  Cron Timer  │────▶│  RSS Feed    │──┐
│  (scheduled) │     │  (any feed)  │  │
└─────────────┘     └──────────────┘  │
                                       ├──▶ Sanitize ──▶ AI Content Factory ──▶ Email Preview ──▶ Sanitize HTML
┌─────────────┐                        │    RSS Input    (7 platforms)          (LLM → HTML)      (XSS protection)
│  Web Form   │────────────────────────┘                                              │
│  (manual)   │                                                                       ▼
└─────────────┘                                                               ┌──────────────┐
                                                                              │ Email Approval│
                                                                              │ (Approve/     │
                                                                              │  Reject)      │
                                                                              └───────┬──────┘
                                                                                      │ ✅
                                                          ┌───────────────────────────┼───────────────────┐
                                                          ▼                           ▼                   ▼
                                                   ┌────────────┐           ┌──────────────┐    ┌─────────────┐
                                                   │ Instagram   │           │ Facebook     │    │ LinkedIn    │
                                                   │ + X/Twitter │           │ Page Post    │    │ Org Post    │
                                                   └────────────┘           └──────────────┘    └─────────────┘
                                                          │
                                                          ▼
                                                   ┌──────────────┐
                                                   │ Results Email │
                                                   │ (status per   │
                                                   │  platform)    │
                                                   └──────────────┘
```

## Platforms Supported

| Platform | Auto-Post | Content Type |
|----------|-----------|-------------|
| **LinkedIn** | ✅ | Image + text (organization page) |
| **Instagram** | ✅ | Image + caption (business account via Graph API) |
| **Facebook** | ✅ | Photo + message (page post) |
| **X (Twitter)** | ✅ | Image + tweet |
| **TikTok** | 📝 | Script/caption generated (manual post) |
| **Threads** | 📝 | Text generated (manual post) |
| **YouTube Shorts** | 📝 | Script/title/description generated (manual post) |

## Features

- **Dual input modes**: Scheduled via cron (pulls from any RSS feed) or manual via web form
- **AI content generation**: GPT-4.1-mini generates platform-optimized posts with structured JSON output
- **AI image generation**: Google Gemini generates brand-consistent illustrations
- **Content categorization**: Posts automatically mapped to cybersecurity education pillars (Backups, Encryption, Security, Updates, Risk Reduction, Education)
- **Human approval gate**: Preview email with approve/reject buttons, 60-minute timeout
- **Multi-platform publishing**: Parallel posting to Instagram, X, Facebook, LinkedIn
- **Results reporting**: Post-publish status email with per-platform success/error details
- **Input sanitization**: RSS feed content stripped of scripts and event handlers before LLM processing
- **Output sanitization**: LLM-generated HTML sanitized before email insertion

## Security Hardening

- **imgbb API key** stored in n8n Credential Manager (Generic Credential Type → Query Auth)
- **RSS input sanitization** strips `<script>`, `<iframe>`, `<style>`, `<object>`, `<embed>`, event handlers, and `javascript:` URIs
- **Email HTML sanitization** removes dangerous elements from LLM-generated approval email content
- **All platform credentials** use n8n's Credential Manager (OAuth2 or API key)
- **No hardcoded secrets** in workflow JSON

## Prerequisites

### n8n Instance
- Self-hosted n8n v2.8+ (tested on v2.8.3)
- Docker Compose recommended

### Required Credentials

| Credential | Type | Used For |
|-----------|------|----------|
| OpenAI API | Native OpenAI | Content generation (GPT-4.1-mini) |
| Google Gemini API | Native Google PaLM | Image generation |
| SerpAPI | Native SerpAPI | Web search tool for AI agent |
| Facebook Graph API | Native Facebook | Instagram + Facebook posting |
| X (Twitter) OAuth2 | Native Twitter | X posting + media upload |
| LinkedIn OAuth2 | Native LinkedIn | LinkedIn organization posting |
| Microsoft Outlook OAuth2 | Native Outlook | Approval + results emails |
| imgbb API | Generic → Query Auth | Temporary image hosting |

### Platform Setup

You'll need:
- **Facebook/Instagram**: Facebook App with `pages_manage_posts`, `instagram_basic`, `instagram_content_publish` permissions. Business account linked.
- **X (Twitter)**: Developer account with OAuth 2.0, `tweet.write` and `media.upload` scopes
- **LinkedIn**: App with `w_member_social` or `w_organization_social` permissions
- **imgbb**: Free API key from [imgbb.com](https://imgbb.com/)
- **SerpAPI**: Free tier key from [serpapi.com](https://serpapi.com/)

## Installation

### 1. Import Workflow

In n8n UI: **Workflows** → **Import from File** → select `social-media-content-factory.json`

### 2. Configure Credentials

Open the workflow and assign credentials to every node with a ⚠️ warning.

### 3. Update Platform IDs

Search the workflow for these placeholders and replace with your actual IDs:

| Placeholder | Where to Find It |
|------------|-------------------|
| `YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID` | Facebook Business Suite → Instagram → Settings |
| `YOUR_FACEBOOK_PAGE_ID` | Facebook Page → About → Page ID |
| `YOUR_LINKEDIN_ORG_ID` | LinkedIn Company Page URL → Admin tools |

These appear in the **Instagram Image**, **Instagram Post**, **Facebook Post**, and **LinkedIn Post** nodes.

### 4. Update Email Recipient

Search for `your-email@example.com` and replace with your approval email address. Found in:
- **Send message and wait for response** (approval email)
- **Send results** (results email)

### 5. Customize RSS Source

Edit the **Worker RSS** node URL to pull from your preferred RSS feed (default: cybersecurity news).

### 6. Customize Content Prompt

The **Social Media Content Factory** agent node contains the full content generation prompt. Customize:
- Brand name and identity
- Content categories (default: cybersecurity education pillars)
- Platform-specific tone and style rules
- Hashtag strategy

### 7. Customize Image Prompt

The **Generate an image** node prompt controls the AI-generated image style. Customize:
- Brand colors
- Visual style preferences
- Text overlay rules

### 8. Activate

1. Test with the manual form first
2. Verify approval email arrives and approve/reject works
3. Confirm posts appear on each platform
4. Then activate for cron scheduling

## Workflow Structure (70 nodes)

```
Step 1: Content Creation
  Cron → RSS Feed → Clean XML → Extract → Sanitize → AI Agent (7-platform JSON)
  Form → Manual Input ─────────────────────┘

Step 2: Image Generation
  AI-generated (Gemini) or user-uploaded → imgbb hosting

Step 3: Approval & Publishing
  LLM formats HTML email → Sanitize HTML → Email approval gate
  → Approved: parallel post to Instagram, X, Facebook, LinkedIn
  → Rejected: workflow stops

Step 4: Results
  Aggregate results → LLM formats status table → Results email
```

## Customization

### Change RSS Source
Edit the **Worker RSS** node URL to pull from any RSS feed.

### Add/Remove Platforms
The structured output schema in **Structured Output Parser** defines which platforms get content. Remove platforms from the schema and disconnect their posting nodes.

### Change Content Categories
Edit the `content_category` enum in the **Structured Output Parser** schema and update the matching instructions in the **Social Media Content Factory** agent prompt.

### Change Approval Timeout
Edit **Send message and wait for response** → Options → Limit Wait Time (default: 60 minutes).

### Change Cron Schedule
Edit the **Cron** node trigger times (default: daily at 7:00 AM).

## License

MIT License — see [LICENSE](LICENSE) for details.
