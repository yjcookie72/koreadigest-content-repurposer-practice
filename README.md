# AI Content Repurposer -- n8n Workflow

An n8n automation workflow that takes a blog post URL and generates platform-specific content for LinkedIn, Twitter/X, Email Newsletter, and Instagram. Powered by OpenAI GPT-4o-mini with configurable prompts and per-platform toggles.

## Use Case

Content creators and marketing teams spend hours manually adapting a single article for different social media platforms. Each platform has its own tone, format, character limits, and engagement patterns. This workflow automates the entire process: paste a URL, get publish-ready content for four platforms in under 30 seconds.

## Architecture

```
                          +------------------+
                          | Webhook / Manual |
                          |    Trigger       |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |   Set Defaults   |
                          | (URL, platforms, |
                          |  tone, handle)   |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |  HTTP Request    |
                          |  (Fetch Article) |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |  HTML Extract    |
                          | (title, body,    |
                          |  meta desc)      |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |   Code Node      |
                          | (Clean text,     |
                          |  extract points, |
                          |  calc read time) |
                          +--------+---------+
                                   |
              +--------------------+--------------------+
              |                    |                    |                    |
     +--------v-------+  +--------v-------+  +--------v-------+  +--------v-------+
     | AI: LinkedIn   |  | AI: Twitter    |  | AI: Newsletter |  | AI: Instagram  |
     | Post Generator |  | Thread Gen     |  | Summary Gen    |  | Caption Gen    |
     +--------+-------+  +--------+-------+  +--------+-------+  +--------+-------+
              |                    |                    |                    |
     +--------v-------+  +--------v-------+  +--------v-------+  +--------v-------+
     | IF: LinkedIn   |  | IF: Twitter    |  | IF: Newsletter |  | IF: Instagram  |
     | Enabled?       |  | Enabled?       |  | Enabled?       |  | Enabled?       |
     +--------+-------+  +--------+-------+  +--------+-------+  +--------+-------+
              |                    |                    |                    |
     +--------v-------+  +--------v-------+  +--------v-------+  +--------v-------+
     | Format LinkedIn|  | Format Twitter |  | Format News-   |  | Format Insta-  |
     | Output         |  | Output         |  | letter Output  |  | gram Output    |
     +--------+-------+  +--------+-------+  +--------+-------+  +--------+-------+
              |                    |                    |                    |
              +--------------------+--------------------+--------------------+
                                   |
                          +--------v---------+
                          |  Merge All       |
                          |  Outputs         |
                          +--------+---------+
                                   |
                          +--------v---------+
                          |  Build Summary   |
                          |  (Code Node)     |
                          +--------+---------+
                                   |
                      +------------+------------+
                      |                         |
             +--------v---------+     +--------v---------+
             | Google Sheets    |     | Slack            |
             | (Log Results)    |     | (Notification)   |
             +--------+---------+     +------------------+
                      |
             +--------v---------+
             | Webhook Response |
             +------------------+
```

## Features

- **Dual trigger**: Webhook (for integrations) and Manual (for testing)
- **Smart text extraction**: HTML parsing with boilerplate removal, key point extraction, and reading time calculation
- **4 AI generation branches** running in parallel for speed
- **Per-platform toggles**: Control which platforms to generate for via the `platforms` array
- **Platform-native prompts**: Each AI node uses a carefully crafted prompt optimized for the target platform's content format
- **Google Sheets logging**: Every run is logged with timestamps, URLs, and all generated content
- **Slack notifications**: Summary of all generated content sent to a channel
- **Webhook response**: Returns a JSON summary to the caller
- **Configurable tone**: Pass `professional`, `conversational`, or `casual` to adjust AI output
- **Author handle injection**: Generated content references the correct social media handle

## Prerequisites

- **n8n** (self-hosted or cloud) -- version 1.0+
- **OpenAI API key** (GPT-4o-mini recommended for cost efficiency)
- **Google Sheets** OAuth2 credentials (for logging)
- **Slack** OAuth2 credentials (for notifications)

## Setup Guide

### 1. Import the Workflow

1. Open your n8n instance
2. Go to **Workflows** > **Import from File**
3. Select `workflow.json` from this repository
4. The workflow will appear with all nodes and connections

### 2. Configure Credentials

You need to set up three credentials in n8n:

**OpenAI API:**
1. Go to **Settings** > **Credentials** > **Add Credential**
2. Search for "OpenAI API"
3. Enter your API key
4. Save -- n8n will assign a credential ID automatically

**Google Sheets OAuth2:**
1. Add a new "Google Sheets OAuth2 API" credential
2. Follow n8n's OAuth flow to authorize your Google account
3. Create a Google Sheet with these column headers in Row 1:
   - `url`, `article_title`, `timestamp`, `platforms_generated`, `linkedin_content`, `twitter_content`, `newsletter_content`, `instagram_content`, `total_characters`, `status`
4. Copy the Sheet ID from the URL and update the "Log to Google Sheets" node

**Slack OAuth2:**
1. Add a new "Slack OAuth2 API" credential
2. Authorize your Slack workspace
3. Update the channel name in the "Send Slack Summary" node (default: `#content-repurposer`)

### 3. Update Node References

After importing, update these nodes with your credential IDs:
- All 4 "AI Generate" nodes: Select your OpenAI credential
- "Log to Google Sheets": Select your Google Sheets credential and set the Sheet ID
- "Send Slack Summary": Select your Slack credential and set the channel

### 4. Activate the Workflow

1. Toggle the workflow to **Active**
2. The webhook URL will be displayed in the Webhook Trigger node
3. Test with a manual execution first

## Usage

### Via Webhook (Production)

```bash
curl -X POST https://your-n8n-instance.com/webhook/repurpose-content \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://blog.hubspot.com/marketing/ai-content-creation",
    "platforms": ["linkedin", "twitter", "newsletter", "instagram"],
    "tone": "professional",
    "author_handle": "@jaikanth"
  }'
```

### Via Manual Trigger (Testing)

1. Open the workflow in the n8n editor
2. Click **Execute Workflow**
3. The "Set Defaults" node will use fallback values (or edit it to set your test URL)

### Payload Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `url` | string | Yes | -- | Full URL of the article to repurpose |
| `platforms` | array | No | All four | Which platforms to generate for |
| `tone` | string | No | `professional` | `professional`, `conversational`, or `casual` |
| `author_handle` | string | No | `@author` | Social media handle for CTAs |

## AI Prompt Customization

Each platform's AI prompt is documented in the `ai-prompts/` directory:

| File | Platform | Key Characteristics |
|------|----------|-------------------|
| `linkedin-prompt.md` | LinkedIn | Hook-body-CTA framework, 1300 char limit, 3-5 hashtags |
| `twitter-prompt.md` | Twitter/X | 5-7 tweet thread, 280 char per tweet, JSON array output |
| `newsletter-prompt.md` | Email | Subject line + preview text + HTML body, JSON output |
| `instagram-prompt.md` | Instagram | Emoji-rich caption, 30 hashtags, carousel slide suggestions |

To customize prompts, edit the `messages` parameter in the corresponding AI node. The prompt files in `ai-prompts/` serve as documentation and reference for the prompts embedded in the workflow nodes.

### Prompt Tips

- Keep system messages short and role-focused
- Put all formatting rules in the user message
- Specify the exact output format (raw text vs JSON)
- Include examples of the expected output
- Set character limits explicitly -- AI models respect these when stated clearly

## Platform Configuration

### LinkedIn (Optional Direct Posting)

To post directly to LinkedIn, add an HTTP Request node after "Format LinkedIn Output":
- Method: POST
- URL: `https://api.linkedin.com/v2/ugcPosts`
- Authentication: OAuth2 with LinkedIn credentials
- Body: UGC Post format with the generated content

### Twitter/X (Optional Direct Posting)

To post threads directly, add a Code node that iterates through the tweet array and posts each via the Twitter API v2:
- Endpoint: `https://api.twitter.com/2/tweets`
- Use `reply.in_reply_to_tweet_id` to chain tweets into a thread

### Email Newsletter

Connect to your email service provider (Mailchimp, ConvertKit, Resend, etc.) using n8n's native nodes or HTTP Request with the provider's API.

### Instagram

Instagram posting requires the Facebook Graph API via a connected Facebook Page. Use the "Facebook Graph API" node in n8n with a Page Access Token.

## Nodes Used

| Node Type | Count | Purpose |
|-----------|-------|---------|
| `n8n-nodes-base.webhook` | 1 | HTTP trigger endpoint |
| `n8n-nodes-base.manualTrigger` | 1 | Manual testing trigger |
| `n8n-nodes-base.set` | 5 | Data transformation and defaults |
| `n8n-nodes-base.httpRequest` | 1 | Fetch article HTML |
| `n8n-nodes-base.html` | 1 | Extract content from HTML |
| `n8n-nodes-base.code` | 2 | Text cleaning + summary building |
| `@n8n/n8n-nodes-langchain.openAi` | 4 | AI content generation (one per platform) |
| `n8n-nodes-base.if` | 4 | Platform enable/disable gates |
| `n8n-nodes-base.merge` | 1 | Combine parallel branch outputs |
| `n8n-nodes-base.googleSheets` | 1 | Log all generated content |
| `n8n-nodes-base.slack` | 1 | Send summary notification |
| `n8n-nodes-base.respondToWebhook` | 1 | Return response to webhook caller |
| **Total** | **23** | |

## Sample Input / Output

See `test-payload.json` for sample webhook payloads and `sample-output.json` for a complete example of all four platform outputs generated from a single article.

## Cost Estimate

Using GPT-4o-mini at current pricing:
- ~4,000 input tokens per platform (article context + prompt)
- ~500 output tokens per platform
- **Total per run: ~18,000 tokens (~$0.003)**
- At 100 articles/month: approximately $0.30/month in AI costs

## License

MIT -- see [LICENSE](LICENSE) for details.
