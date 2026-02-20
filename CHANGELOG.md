# Changelog

## [1.0.0] - 2025-02-20

### Security Hardening
- Moved imgbb API key from hardcoded query parameter to n8n Credential Manager (Generic Credential Type → Query Auth)
- Added **Sanitize RSS Input** Code node to strip `<script>`, `<iframe>`, `<style>`, `<object>`, `<embed>`, event handlers, and `javascript:` URIs from RSS feed content before LLM processing
- Added **Sanitize Email HTML** Code node to remove dangerous HTML elements from LLM-generated approval email content
- Scrubbed all credential IDs, webhook IDs, platform IDs, email addresses, and instance fingerprint from exported JSON

### Bug Fixes
- Fixed typo: "Instragram Post" → "Instagram Post" (node name and all expression references)
- Removed disconnected orphan node
- Cleared pinned test data from Cron trigger node

### Content Generation
- AI image generation using Google Gemini 2.5 Flash Image model
- Comprehensive content categorization prompt with topic analysis, audience detection, and image tone matching
- Structured JSON output schema covering 7 platforms with per-platform formatting rules
- X-Twitter character limit enforcement (280 chars, hashtags in separate array)

### Workflow
- Dual input: Cron + RSS feed or manual web form
- Email approval gate with 60-minute timeout via Microsoft Outlook
- Parallel publishing to Instagram, X, Facebook, LinkedIn
- Post-publish results summary email with per-platform status table
