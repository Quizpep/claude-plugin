# Quizpep for Claude

Build Shopify product quizzes by chatting with Claude. Describe the quiz you want, and Claude builds it
in [Quizpep](https://quizpep.com) with your store's real products: it writes the questions, matches
every answer to products, checks the quiz and test-runs it, then saves it as a draft for you to
preview. It only publishes when you ask.

> "Build a 5-question skincare quiz that recommends 3 products, asks for an email at the end and
> offers 10% off. Use our brand colors."

You need the [Quizpep app](https://apps.shopify.com/quizpep-product-quiz-builder) installed on your
Shopify store. Building and editing quizzes works on every plan; publishing needs the Scale plan
(development stores are free).

## Install

### Claude Code

```
/plugin marketplace add quizpep/claude-plugin
/plugin install quizpep@quizpep
```

Then run `/mcp`, choose **quizpep** and sign in: enter your store, and click **Allow** in your
Shopify admin.

### Claude (website and desktop app)

1. Settings → Connectors → **Add custom connector**. Name: `Quizpep`. URL: `https://app.quizpep.com/mcp`.
2. Click **Connect**, enter your store, and click **Allow** in your Shopify admin.
3. Optional, for better quizzes: Settings → Capabilities → Skills → upload
   [`quizpep-quiz-builder.zip`](https://quizpep.com/claude/quizpep-quiz-builder.zip).

## What's inside

| Part | What it does |
|---|---|
| `.mcp.json` | The Quizpep connector (`https://app.quizpep.com/mcp`): Claude's access to your store's quizzes and product catalog. |
| `skills/quizpep-quiz-builder` | How to build a good Quizpep quiz: the workflow, the quiz format, worked examples and a quality checklist. |

### Connector tools

`get_store`, `search_products`, `list_templates`, `get_template`, `get_quiz_format`, `list_quizzes`,
`get_quiz`, `validate_quiz`, `create_quiz`, `update_quiz`, `test_quiz`, `upload_image`, `publish_quiz`.

## Privacy and safety

- Each connection is approved by staff in the store's own Shopify admin, and sees only that store.
- Claude can read products, tags and collections, and create and change quizzes. It can't see
  customers, orders or quiz answers, and it can't delete quizzes.
- Quizzes are saved as drafts. Publishing happens only on request, and only on a plan that includes it.
- Disconnect anytime in Quizpep → Integrations → Claude. Uninstalling Quizpep ends every connection.

Help: [quizpep.com/claude](https://quizpep.com/claude) · support@quizpep.com
