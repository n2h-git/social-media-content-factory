# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

AI-powered multi-platform social media content generation and publishing system built as an n8n workflow. Generates platform-optimized posts from RSS feeds or manual input, creates AI images, gates on human email approval, then auto-publishes to Instagram, X, Facebook, and LinkedIn.

## Tech Stack

- **Workflow Engine:** n8n v2.8+ (self-hosted Docker)
- **Content LLM:** OpenAI GPT-4.1-mini
- **Image Generation:** Google Gemini 2.5 Flash (temperature 0.4)
- **Image Hosting:** imgbb API (24-hour expiry)
- **Web Search:** SerpAPI (tool for content agent)
- **Approval/Notifications:** Microsoft Outlook OAuth2
- **Auto-Post Platforms:** Instagram (Graph API v23.0), X/Twitter (OAuth2), Facebook (Graph API), LinkedIn (OAuth2)
- **Manual Platforms:** TikTok, Threads, YouTube Shorts (content generated, posting manual)

## Workflow Architecture

```
Dual Input → Sanitize → AI Content Factory → AI Image Gen → Email Approval Gate → Parallel Publishing → Results Email
```

**Dual Input Modes:**
1. **Cron + RSS** — scheduled daily at 7 AM, fetches from RSS feed (default: BleepingComputer.com)
2. **Web Form** — manual topic/keywords/links entry via n8n formTrigger

**Content Flow (70+ nodes):**
1. Input triggers merge with source tag (cron vs form)
2. RSS input sanitized (XSS: scripts, iframes, event handlers, javascript: URIs removed)
3. AI Agent (GPT-4.1-mini) generates structured JSON for 7 platforms via Structured Output Parser
4. Google Gemini generates brand-consistent illustration, hosted on imgbb
5. LLM formats HTML approval email, output sanitized for XSS
6. Outlook sends approval email with 60-minute timeout
7. If approved: parallel posts to Instagram, X, Facebook, LinkedIn
8. Results email sent with per-platform success/error status table

## Structured Output Schema

The AI agent outputs JSON with: `name`, `description`, `image_prompt`, `content_category` (one of: Backups, Encryption, Security, Updates, Risk Reduction, Education), and `platform_posts` containing per-platform objects (LinkedIn, Instagram, Facebook, X-Twitter, TikTok, Threads, YouTube_Shorts) each with post/caption text, hashtags, image/video suggestions, and CTAs.

## Credentials Required (8 total, all in n8n Credential Manager)

| Credential | n8n Type | Purpose |
|-----------|----------|---------|
| OpenAI API | Native OpenAI | GPT-4.1-mini content generation (3 instances) |
| Google Gemini API | Native Google PaLM | Image generation + email HTML formatting |
| SerpAPI | Native SerpAPI | Web search tool for content agent |
| Facebook Graph API | Native Facebook | Instagram + Facebook posting |
| X OAuth2 | Native Twitter OAuth2 | X posting + media upload |
| LinkedIn OAuth2 | Native LinkedIn | LinkedIn org page posting |
| Microsoft Outlook OAuth2 | Native Outlook | Approval + results emails |
| imgbb API | Generic Query Auth | Image hosting |

## Setup After Import

1. Import `workflows/social-media-content-factory.json` into n8n
2. Assign credentials to all nodes with warning icons
3. Replace placeholder IDs:
   - `YOUR_INSTAGRAM_BUSINESS_ACCOUNT_ID` in Instagram nodes
   - `YOUR_FACEBOOK_PAGE_ID` in Facebook Post node
   - `YOUR_LINKEDIN_ORG_ID` in LinkedIn Post node
   - `your-email@example.com` in approval + results email nodes
4. Customize RSS source URL in "Worker RSS" node (default: BleepingComputer)
5. Customize content categories and tone in "Social Media Content Factory" agent prompt
6. Test with manual form first, then activate cron schedule

## Security

- Zero hardcoded secrets — all credentials in n8n Credential Manager
- RSS input sanitized before LLM (removes script/iframe/style/object/embed tags, event handlers, javascript: and data: URIs)
- LLM-generated email HTML sanitized before Outlook insertion (same XSS protections)
- 60-minute approval timeout prevents stale approvals

## Key Files

- `workflows/social-media-content-factory.json` — the complete n8n workflow (75KB, 70+ nodes)
- `README.md` — full setup guide with credential table and customization instructions
- `CHANGELOG.md` — v1.0.0 release notes
